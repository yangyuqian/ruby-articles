# 引言

采用 Ruby 开发比较小的项目时, Ruby 内核类加载机制就够用了. 然而随着项目发展,类结构变得越来越复杂, Ruby 内核的类加载机制显得有些力不从心.

在 Ruby 中，Class/Module 都是以常量（Contant）的方式进行管理的. 而在解释器中 Module/Class 之间的嵌套关系可以简单的看作一个比较大的 Hash, 里面维护了顶层 Object 类到下面的所有 Module/Class 直接的引用关系，如：

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

Ruby 内核已经提供了找到这个类的一些机制(在后面的章节会提到), 那问题来了，类结构比较复杂的情况下，怎么去保证一个正确的 User 类在被使用到的时候已经加载了呢？要知道类结构比较复杂的情况下，Ruby 内核类加载需要开发人员去 “人肉” 维护整个类树，这显然是个伪命题.

这时就有一些乐于给自己挖坑的 Technicians 想到了一个突破口: 重写 Ruby 的 const_missing 方法:

```
module A
  def self.const_missing(name)
    # Find name, and load name
  end

  B
end
```

显然在 A 用到 B 的时候它还没有被加载, 解释器发现找不到 B 就跑去告诉 const_missing 方法，说: “Find it, and load it”. 

然后? 是不是就没有然后了？因为似乎问题已经解决了: const_missing 高兴的加载了类（先别管怎么加载的），然后解释器就开心用这个类去做一些羞羞的事情.

当然不是！！！首先从 Ruby 内核类加载的角度来看, 不论怎么去加载一个类，一定有一个潜在的条件: 文件的位置已知.

试想如果能自动加载，而我们却需要为自动加载“人肉”维护一个大的 Hash 表, 告诉解释器应该加载那个文件, 这个结果和直接用 Ruby 内核类加载就没本质区别了, 这位可怜的技术人员没能降低维护成本，又绕回来了！

Rails 的出现让基于 Ruby 来做大项目成为可能, 我相信原因之一是因为 Rails 解决了以上提到的问题. 

本文从一些实际问题入手，结合一些实例，从以下方面介绍 Rails 提供的类加载解决方案：

1. Ruby 内核类加载机制

2. Ruby “常量查找”(Constant Lookup)机制

3. Rails 类加载机制


# Ruby 内核类加载机制

所谓“万法归宗”，不论怎么高大上的解决方案，落实到实现上，也是采用了一些已有的东西，Ruby 的内核加载机制就是万法的源头.

Ruby 内核类加载机制已经提供了类加载所需要的所有能力, 具体参见 [Ruby 内核类加载机制](https://github.com/yangyuqian/ruby-articles/blob/master/RUBY-KERNEL-CLASS-LOADER.md), 而Rails等框架提供的能力就是用“启发式”的去找到一个类定义的文件, 并将其自动加载到内存中, 可见Ruby 和 Rails 珠联璧合、相得益彰.

# Ruby “常量查找”(Constant Lookup)机制

“启发式”的类加载方式需要有一个触发点来告诉响应的框架什么时候加载什么类: 这个触发点就是 Ruby 的 const_missing 方法, 而在什么时候会触发它呢？这就需要了解 Ruby 是怎么判断一个类（常量）是否已经在内存中定义了的原理了.

详情参见 [Ruby “常量查找”(Constant Lookup)机制](https://github.com/yangyuqian/ruby-articles/blob/master/RUBY-CONSTANT-LOOKUP.md)

# Rails 类加载机制

前面做了这么多准备，就为了 Rails 这一哆嗦，本章主要介绍 Rails 中给出的“启发式”类加载解决方案。

值得重复说明的是: Rails 提供的是一种“启发式”的查找类定义文件的算法，真正找到之后加载用的还是 Ruby Kernel 提供的加载机制，具体方式和 RAILS_ENV 有关(dev 下用 load, prod 下用 require).

基本流程是，解释器会先用 Ruby 内核的常量查找算法尝试去找一个类定义，找不到就告诉const_missing，然后 Rails 就跑去猜这个常量会定义在哪里，如果 Rails 还找不到，就抛异常.

## autoload_paths

Rails 维护了类似于 $LOAD_PATH 的变量 autoload_paths，Rails 3 中默认会将 app 下的子目录以及 lib 目录全部加入 autoload_paths, Rails 4 中去掉了 lib, 不过我们可以自己加上去:

```
# config/application.rb
config.autoload_paths << "#{Rails.root}/lib"
```

autoload_paths 实际上是在 ActiveSupport::Dependencies.autoload_paths 维护的，Rails 中的 autoload 机制本质上是 ActiveSupport 的 autoload 机制.

以下代码将当前目录加入 autoload_paths, 这样在 Ruby 找不到某个常量定义的时候，ActiveSupport 就会跑去找到定义的文件并自动加载（脱离 Rails 调试 ActiveSupport需要了解一些 [Bundler相关的知识](https://github.com/yangyuqian/ruby-articles/blob/master/BUNDLER.md)）.

```
ActiveSupport::Dependencies.autoload_paths << '.'
```

## ActiveSupport 中的常量查找算法

Rails 中仅仅提供了一个 autoload_paths 的配置入口，实际的内容都在 ActiveSupport 中维护，因此脱离 Rails 谈 Rails 的类加载机制可能更清楚一些.

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

开一个 irb console, 用 load 命令手动加载 demo/role.rb，发现即使没有为 demo/user.rb 声明任何的 load/require, 它居然被正确访问到了! 这就是 Rails 中的 autoload 机制.

Rails 中假设我们的类名和文件名是有直接关系的，例如 app/models/auth/user.rb 中就应该是以下定义:

```
# app/models/auth/user.rb
module Auth
  class User
  end
end
```

Rails(ActiveSupport) 中任意常量 C 的查找算法伪代码:

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

### 线程安全

[Eager loading for greater good](http://blog.plataformatec.com.br/2012/08/eager-loading-for-greater-good)一文中提到 Rails 3 下的 autoload 是线程不安全的，为什么呢?

Rails 的 autoload 是基于 Ruby 内核常量查找机制的，其无法获取 nesting 内容，触发 const_missing 的位置在哪儿，Rails 就获得了那个 missing 的常量，具体加载的类和实际执行的路径有关，存在线程安全隐患，考虑下面的例子:

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
    
    * ActiveSupport 毕竟只是苦力的干活，Ruby 内核才是掌柜的干活，苦力肯定不能自作去加载一个 Ruby 没有加载的东西
    
    * 无奈之下，ActiveSupport 也只好说自己没找到这个常量（实际上是找到了，但一切以 Ruby 的判断为准！N 个凡是！）


### autoload 目录下不要用 require



# 参考文献

[Autoloading and Reloading Constants](http://guides.rubyonrails.org/autoloading_and_reloading_constants.html)

[Eager loading for greater good](http://blog.plataformatec.com.br/2012/08/eager-loading-for-greater-good)

[Rails autoloading — how it works, and when it doesn't](http://urbanautomaton.com/blog/2013/08/27/rails-autoloading-hell)
