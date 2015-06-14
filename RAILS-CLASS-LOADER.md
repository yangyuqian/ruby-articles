# 引言

Rails 的出现让基于 Ruby 来做大项目成为可能, 原因之一是其巧妙地解决了 Ruby 中复杂的类加载问题. 

Ruby 中 Class/Module 都是以常量（Contant）的方式进行管理的. 而在解释器中 Module/Class 之间的嵌套关系可以简单的看作一个比较大的 Hash, 里面维护了顶层 Object 类到下面的所有 Module/Class 直接的引用关系，如：

```
module Auth
  class User
  end
end
```
可以抽象成以下的 Hash 表:

```
{
  'Object': {
    'Auth::User': User
  }
}
```

这让我们在代码的其他地方访问 User 类成为可能:

```
module Auth
  module Role
    User
  end
end
```

内核已经提供了查找类常量的机制(在后面的章节会提到). 在类结构比较复杂时，怎么去保证一个正确的 User 类在被使用到的时候已经加载了呢? 小项目中可以人工维护这个依赖加载, 在项目中如果还让开发人员去 “人肉” 维护整个类依赖树, 这将是个伪命题.

这时有人想到了一个突破口: 重写 Ruby 的 const_missing 方法:

```
module A
  def self.const_missing(name)
    # Find name, and load name
  end

  B
end
```

如果 A 用到 B 的时候它还没有被加载, 解释器发现找不到 B 就跑去告诉 const_missing 方法，说: “Find it, and load it”. 

然后? 是不是就没有然后了？因为似乎问题已经解决了: const_missing 高兴的加载了类（先别管怎么加载的）, 然后解释器就开心用这个类去做一些羞羞的事情.

当然不是！！！首先从内核类加载的角度来看, 不论怎么去加载一个类，一定有一个潜在的条件: 文件的位置已知.

试想如果能自动加载，而我们却需要为自动加载“人肉”维护一个大的 Hash 表, 告诉解释器应该加载哪个文件, 结果和直接用内核类加载就没本质区别了, 又绕回来了！

本文从一些实例代码入手, 从以下方面介绍 Rails 提供的类加载解决方案：

1. Ruby 内核类加载机制

2. Ruby “常量查找”(Constant Lookup)机制

3. Rails 类加载机制


# Ruby 内核类加载机制

Ruby 内核类加载机制已经提供了类加载所需要的所有能力, 具体参见 [Ruby 内核类加载机制](https://github.com/yangyuqian/ruby-articles/blob/master/RUBY-KERNEL-CLASS-LOADER.md), 而Rails等框架提供的能力就是用“启发式”的去找到一个类定义的文件的位置, 将其自动加载到内存中.

# Ruby “常量查找”(Constant Lookup)机制

“启发式”的类加载方式需要有一个触发点来告诉 Rails 什么时候加载什么类: 这个触发点就是 Ruby 的 const_missing 方法, 而在什么时候会触发它呢? 这就需要了解 Ruby 是怎么判断一个类（常量）是否已经在内存中定义了的原理了, 即 [Ruby 中的常量查找](https://github.com/yangyuqian/ruby-articles/blob/master/RUBY-CONSTANT-LOOKUP.md).

# Rails 类加载机制

本章主要介绍 Rails 中给出的“启发式”类加载解决方案。

值得强调的是: Rails 提供的是一种“启发式”的查找类定义文件位置的算法，真正找到之后加载用的还是 Ruby Kernel 提供的加载机制，具体方式和 RAILS_ENV 有关(dev 下用 load, prod 下用 require).

基本流程是，解释器会先用 Ruby 内核的常量查找算法尝试去找一个类定义，找不到就告诉const_missing，然后 Rails 就跑去猜这个常量会定义在哪里，如果 Rails 还找不到，就抛异常.

## autoload_paths

Rails 维护了类似于 $LOAD_PATH 的变量 autoload_paths，Rails 3 中默认会将 app 下的子目录以及 lib 目录全部加入 autoload_paths, Rails 4 中去掉了 lib. 

以下是一个刚升成的 Rails项目的 autoload 路径:

```
$ bin/rails r 'puts ActiveSupport::Dependencies.autoload_paths'
.../app/assets
.../app/controllers
.../app/helpers
.../app/mailers
.../app/models
.../app/controllers/concerns
.../app/models/concerns
.../test/mailers/previews
.../lib
```

在 Rails 中还可以添加自定义的 autoload 路径:

```
# config/application.rb
config.autoload_paths << "#{Rails.root}/something"
```

autoload_paths 实际上是在 ActiveSupport::Dependencies.autoload_paths 中定义的, 这是一个 String Array, 可见 Rails 的 autoload 本质上是 ActiveSupport 的 autoload 机制. 本文中刻意淡化 Rails 中的一些 autoload 相关的配置入口, 转而从 ActiveSupport 的一些接口入手, 介绍 Rails 中的类加载. 脱离 Rails 调试 ActiveSupport需要了解一些 [Bundler相关的知识](https://github.com/yangyuqian/ruby-articles/blob/master/BUNDLER.md).

以下代码将当前目录加入 autoload_paths, 这样在 Ruby 找不到某个常量定义的时候，ActiveSupport 就会尝试找到常量定义文件并自动加载.

```
ActiveSupport::Dependencies.autoload_paths << '.'
```

## ActiveSupport 中的常量查找算法

新建文件 demo/user.rb，定义如下:

```
# demo/user.rb 
module Demo
  class User
    "class Demo::User loaded"
  end
end
```

将当前目录加入 autoload_paths:

```
2.1.5 :008 >   ActiveSupport::Dependencies.autoload_paths << "."
 => ["."] 
```

新建 demo/role.rb，定义如下:

```
# demo/role.rb
module Demo
  class Role
    User
  end
end
```

开一个 irb console, 用 load 命令手动加载 demo/role.rb，发现即使没有为 demo/user.rb 声明任何的 load/require, 它居然这么顺利成章的自动加载了!

Rails 中假设类常量名和文件名是有直接关系的，例如 app/models/auth/user.rb 中就应该是以下定义:

```
# app/models/auth/user.rb
module Auth
  class User
  end
end
```

Rails(ActiveSupport) 中的 autoload 会根据触发 const_missing 的常量名称来猜测并尝试加载对应的文件, 以加载前例中的 Auth::User 为例:

1. Demo::Role 找不到 User 常量, 触发 const_missing(const_name), 此处 const_name == 'User'

2. ActiveSupport 中先拼接出来一个查询的起点 "#{Demo::Role.name}::#{const_name}", 即 Demo::Role::User

3. 首先尝试查找 autoload_paths 下的 demo/role/user.rb, 没找到

4. 然后往上走一层, 尝试查找 autoload_paths 下的 demo/user.rb, 找到了

如果加载了 demo/user.rb 发现 Demo::User 还是未定义的状态, Rails 默认会抛 LoadError 异常:

```
LoadError: Unable to autoload constant Demo::User, expected ./demo/user.rb to define it
```

具体算法的伪码如下:

```
if the class or module in which C is missing is Object
  let ns = ''
else
  let M = the class or module in which C is missing
 
  if M is anonymous
    let ns = ''
  else
    let ns = M.name
  end
end
 
loop do
  # Look for a regular file.
  for dir in autoload_paths
    if the file "#{dir}/#{ns.underscore}/c.rb" exists
      load/require "#{dir}/#{ns.underscore}/c.rb"
 
      if C is now defined
        return
      else
        raise LoadError
      end
    end
  end
 
  # Look for an automatic module.
  for dir in autoload_paths
    if the directory "#{dir}/#{ns.underscore}/c" exists
      if ns is an empty string
        let C = Module.new in Object and return
      else
        let C = Module.new in ns.constantize and return
      end
    end
  end
 
  if ns is empty
    # We reached the top-level without finding the constant.
    raise NameError
  else
    if C exists in any of the parent namespaces
      # Qualified constants heuristic.
      raise NameError
    else
      # Try again in the parent namespace.
      let ns = the parent namespace of ns and retry
    end
  end
end
```

## 常见误区

### autoload 目录下不要用 require

没有 ActiveSupport 的情况下，我们需要用 Ruby 内核类加载机制来加载依赖:

```
require 'something'
```

这样干的问题主要是 require 只加载一次 something 类, 且只在第一次被加载，这样就永远不会触发 ActiveSupport 的 autoload，这就意味着，调试的时候如果修改了 something 需要重启整个 App.

Rails 中推荐用 RAILS_ENV 来控制类加载方式, 即 development 下用 load, production 下用 require.

### 并发安全

考虑为航天器建模, 先定一个默认的“会飞”的模型:

```
# app/models/flight_model.rb
class FlightModel
end
```

定义一种飞机模型:

```
# app/models/bell_x1/flight_model.rb
module BellX1
  class FlightModel < FlightModel
  end
end
 
# app/models/bell_x1/aircraft.rb
module BellX1
  class Aircraft
    def initialize
      @flight_model = FlightModel.new
    end
  end
end
```

这里的 FlightModel.new 应该是想去新建一个 BellX1::FlightModel, 但如果 FlightModel 已经被加载过, 解释器就能找到一个合法的类定义, 不出发 Rails autoload, BellX1::FlightModel 将永远不可达. 这种情况下代码的结果和具体的执行路径有关, 存在并发安全问题.

### nesting 和 autoload 矛盾

Rails 的 autoload 是基于 Ruby 内核常量查找机制的, 其无法获取 nesting 内容, 具体加载的类活着常量和实际执行的时机有关, 考虑下面的例子:

```
# qux.rb
Qux = "I'm at the root!"

# foo.rb
module Foo
end

# foo/qux.rb
module Foo
  Qux = "I'm in Foo!"
end

# foo/bar.rb
class Foo::Bar
  def self.print_qux
    puts Qux
  end
end
```

在 console 中执行 2 次 Foo::Bar.print_qux, 解释器抛异常:

```
2.1.5 :011 >   Foo::Bar.print_qux
I'm in Foo!
 => nil 
2.1.5 :012 > Foo::Bar.print_qux
NameError: uninitialized constant Foo::Bar::Qux
  from /vagrant/demo1/foo/bar.rb:3:in print_qux'
  from (irb):12
  from /home/vagrant/.rvm/rubies/ruby-2.1.5/bin/irb:11:in <main>'
```

出现这种问题的大致原理:

执行第一次的时候Ruby 尝试去取 Foo::Bar::Qux 发现找不到:

  * 然后找 Object::Qux, 还是找不到

  * 触发 const_missing 交给  ActiveSupport 处理

  * 通过 ActiveSupport 访问 Foo::Bar::Qux:

    * 加载顶层 module, foo.rb

    * 找 Foo::Bar::Qux 找不到

    * 加载 foo/qux.rb

    * 找 Foo::Qux

    * 很开心的加载了 Foo::Qux

执行第二次访问还是走一样的路径，区别在于找 Foo::Qux 的时候发现已经有定义了，就不往上找了
  
  * 但已经存在的 Foo::Qux 并非是一个 missing 的常量，这里就出现了一个悖论：
    
    * Ruby 内核告诉 ActiveSupport 我有一个常量找不到，可能是没加载文件，请帮我找到那个文件并加载这个常量
    
    * 然后 ActiveSupport 开始努力去寻找这个常量，结果找了半天只找到一个已经存在的常量
    
    * ActiveSupport 毕竟只是苦力的干活，Ruby 内核才是掌柜的干活，苦力肯定不能擅作主张去加载一个 Ruby 没有加载的东西（已经存在的常量，如果 Ruby 没有加载就说明不是内核要的那个）
    
    * 无奈之下，ActiveSupport 也只好说自己没找到这个常量（实际上是找到了，但一切以 Ruby 的判断为准！N 个凡是！）


### 不要再 App 启动的时候去 autoload 常量

考虑以下常量赋值:

```
# config/initializers/set_auth_service.rb:
AUTH_SERVICE = if Rails.env.production?
  RealAuthService
else
  MockedAuthService
end
```

Rails 初始化时候的常量是不会被修改的, 之后就算我们修改了对应的 AuthService 实现, 也必须重启才会生效. 采用实现一个动态的 AccessPoint 可以解决这种问题:

```
# app/models/auth_service.rb
class AuthService
  if Rails.env.production?
    def self.instance
      RealAuthService
    end
  else
    def self.instance
      MockedAuthService
    end
  end
end
```
AuthService 是可以动态加载的, 也能够随时被重新加载.

# 参考文献

[Autoloading and Reloading Constants](http://guides.rubyonrails.org/autoloading_and_reloading_constants.html)

[Eager loading for greater good](http://blog.plataformatec.com.br/2012/08/eager-loading-for-greater-good)

[Rails autoloading — how it works, and when it doesn't](http://urbanautomaton.com/blog/2013/08/27/rails-autoloading-hell)

[Configuring Rails Applications](http://guides.rubyonrails.org/configuring.html)