# 引言

本文从一些实例代码入手, 尽可能全面地介绍了 Rails 类加载及涉及的相关技术知识：

1. [Ruby 内核类加载](https://github.com/yangyuqian/ruby-articles/blob/master/RUBY-KERNEL-CLASS-LOADER.md)

2. [Ruby 常量查找](https://github.com/yangyuqian/ruby-articles/blob/master/RUBY-CONSTANT-LOOKUP.md)

3. [ActiveSupport 对内核类加载的扩展](https://github.com/yangyuqian/ruby-articles/blob/master/RAILS-CLASS-LOADER.md#activesupport-对内核类加载的扩展)

4. [Rails 自动类加载](https://github.com/yangyuqian/ruby-articles/blob/master/RAILS-CLASS-LOADER.md#rails-自动类加载)

5. [Rails 类加载中的常见误区](https://github.com/yangyuqian/ruby-articles/blob/master/RAILS-CLASS-LOADER.md#rails-类加载中的常见误区)

# Ruby 内核类加载

Ruby 内核类加载机制已经提供了类加载所需要的所有能力, 具体参见 [Ruby 内核类加载机制](https://github.com/yangyuqian/ruby-articles/blob/master/RUBY-KERNEL-CLASS-LOADER.md), 而Rails等框架提供的能力就是用“启发式”的去找到一个类定义的文件的位置, 将其自动加载到内存中.

# Ruby “常量查找”(Constant Lookup)

“启发式”的类加载方式需要有一个触发点来告诉 Rails 什么时候加载什么类: 这个触发点就是 Ruby 的 const_missing 方法, 而在什么时候会触发它呢? 这就需要了解 Ruby 是怎么判断一个类（常量）是否已经在内存中定义了的原理了, 即 [Ruby 中的常量查找](https://github.com/yangyuqian/ruby-articles/blob/master/RUBY-CONSTANT-LOOKUP.md).

# ActiveSupport 对内核类加载的扩展

## autoload 扩展

ActiveSupport 扩展内核的 autoload, 便于创建简洁的 library (Gem).

比较大的项目中, 通常会将通用模块封装成 Gem, 比如我们可以将 sso 相关的功能封装成 Gem, 目录结构如下:

```
lib/
  sso.rb
  sso
    form_auth.rb
    api_auth.rb
sso.gemspec
```

假设这里提供了2种认证方式, 即基于Form的登陆方式, 在 form_auth.rb 中实现; 此外还能通过 api 登陆, 实现在 api_auth.rb 中.

将整个 Gem 封装好之后，怎么来使用其中的实现呢? 类加载的方式是第一个需要解决的问题.

内核类加载中提供了这种能力, lib/sso.rb 中为整个 Gem 提供了单一的入口, 可以承担加载类的职责, 可以这样实现:

```
# lib/sso.rb

require 'sso/form_auth'
require 'sso/api_auth.rb'
```

在 Rails App 代码中只需要 require 'sso' 就能将整个 sso 库提供的功能引入到项目里面来了. 可见, 这里的 lib/sso.rb 不会有具体的实现代码, 只是一个类加载的入口, 如果想把 sso 名字改成 account 怎么办? 这里给一个完全基于内核类加载的解决方案:

1. 修改目录名和入口文件名:

```
lib/
  account.rb
  account
    form_auth.rb
    api_auth.rb
```

2. 修改 account.rb 中文件路径

```
# lib/account.rb

require 'account/form_auth'
require 'account/api_auth.rb'
```

以上的例子相对简单, 如果要用这种方式管理很多复杂的 Gem 的时候就显得拙荆见肘了. 考虑到很多 Ruby 相关的应用场景都和 Rails 有关，那么能不能更优雅一点呢? 这里有一个信息没有充分利用，那就是 lib/sso.rb 的文件名. 在 ActiveSupport::Dependencies::Autoload 中针对这样的使用场景做了一些扩展:

```
# active_support/dependencies/autoload

def autoload(const_name, path = @_at_path)
  unless path
    full = [name, @_under_path, const_name.to_s].compact.join("::")
    path = Inflector.underscore(full)
  end

  if @_eager_autoload
    @_autoloads[const_name] = path
  end

  super const_name, path
end
```

基于扩展后的 autoload 就可以这样实现前面的 sso 类库的入口:

```
# lib/sso.rb

module Sso
  extend ActiveSupport::Autoload

  autoload :FormAuth
  autoload :ApiAuth 
end
```

入口文件中不在包含相对路径, 只需要给出一个实现的类名即可. 降低了和底层的耦合同时也提升了可读性. 现在要改这个类库的名字只需要改 lib/sso.rb的文件名、module 的名字以及 lib/sso 目录的名字, 是一个 O(1) 复杂度的算法, 相比内核类加载那种方案的 O(n) 算法来说优化了很多.


## eager_autoload

需要考虑线程安全的场景下, 可以使用 require 来将类直接加载至内存中, 但 require 也有和 autoload 一样的问题: 需要指定文件路径. 于是 ActiveSupport 给出了eager_autoload, 可以选择性地加载类库中一些特定的类，避免了使用 require:

```
module MyLib
  extend ActiveSupport::Autoload

  autoload :Model

  eager_autoload do
    autoload :Cache
  end
end
```

这样类库就提供了线程安全和非线程安全两种加载模式, 前者中可以直接在 App 启动的时候执行:

```
MyLib.eager_load!
```

ActiveSupport 中的 eager_autoload 仅仅是记录了需要 eager_load 的文件路径, 只有在执行了 **eager_load!** 才会正真加载:

```
# active_support/dependencies/autoload

def eager_autoload
  old_eager, @_eager_autoload = @_eager_autoload, true
  yield
ensure
  @_eager_autoload = old_eager
end

def eager_load!
  @_autoloads.each_value { |file| require file }
end
```

# Rails 自动类加载

本章主要介绍 Rails 中给出的自动类加载解决方案, 这是一个“启发式”类加载算法。

值得强调的是: 自动类加载是一种“启发式”的查找类定义文件位置的算法，真正找到之后加载用的还是 Ruby Kernel 提供的加载机制，具体方式和 RAILS_ENV 有关(dev 下用 load, prod 下用 require).

基本流程是，解释器会先用 Ruby 内核的常量查找算法尝试去找一个类定义，找不到就告诉const_missing，然后 Rails 就跑去猜这个常量会定义在哪里，如果 Rails 还找不到，就抛异常.

## autoload_paths

Rails 维护了类似于内核中 $LOAD_PATH 的变量 autoload_paths，Rails 3 中默认会将 app 下的子目录以及 lib 目录全部加入 autoload_paths, Rails 4 中去掉了 lib. 

以下是一个刚生成的 Rails 3 项目的 autoload 路径:

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

Rails(ActiveSupport) 中的会根据触发 const_missing 的常量名称来猜测并尝试加载对应的文件, 以加载前例中的 Auth::User 为例:

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

## Rails 类加载中的常见误区

### autoload 目录下不要用 require

没有 ActiveSupport 的情况下，我们需要用 Ruby 内核类加载机制来加载依赖:

```
require 'something'
```

这样干的问题主要是 require 只加载一次 something 类, 且只在第一次被加载，这样就永远不会触发 ActiveSupport 的 autoload，这就意味着，调试的时候如果修改了 something 需要重启整个 App.

Rails 中推荐用 RAILS_ENV 来控制类加载方式, 即 development 下用 load, production 下用 require.

### 并发安全

考虑为航天器建模, 先定一个默认的飞行器的模型:

```
# app/models/flight_model.rb
class FlightModel
end
```

然后定义一种飞机模型，并生产一架飞机(new 一个对象):

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

这里的 FlightModel.new 应该是想去新建一个 BellX1::FlightModel, 但如果 FlightModel 已经被加载过, 解释器就能找到一个合法的类定义, 不触发 Rails autoload, 则 BellX1::FlightModel 不会被加载, 永远不可达. 这种情况下代码的结果和具体的执行路径有关, 存在并发安全问题.

### nesting 和 autoload 矛盾

Rails 的 autoload 是基于 Ruby 内核常量查找机制的, 其无法获取 nesting 内容, 具体加载的类或常量和实际执行的时机有关, 考虑下面的例子:

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

### 不要在 App 启动的时候去 autoload 常量

考虑以下常量赋值:

```
# config/initializers/set_auth_service.rb:
AUTH_SERVICE = if Rails.env.production?
  RealAuthService
else
  MockedAuthService
end
```

Rails 初始化时候的常量是不会被修改的, 之后就算我们修改了对应的 AuthService 实现, 也必须重启才会生效. 实现一个动态的 AccessPoint 可以解决这种问题:

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

# 参考文献

[Autoloading and Reloading Constants](http://guides.rubyonrails.org/autoloading_and_reloading_constants.html)

[Eager loading for greater good](http://blog.plataformatec.com.br/2012/08/eager-loading-for-greater-good)

[Rails autoloading — how it works, and when it doesn't](http://urbanautomaton.com/blog/2013/08/27/rails-autoloading-hell)

[Configuring Rails Applications](http://guides.rubyonrails.org/configuring.html)