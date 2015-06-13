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

## 常见误区



# 参考文献

[Autoloading and Reloading Constants](http://guides.rubyonrails.org/autoloading_and_reloading_constants.html)

[Eager loading for greater good](http://blog.plataformatec.com.br/2012/08/eager-loading-for-greater-good)

[Rails autoloading — how it works, and when it doesn't](http://urbanautomaton.com/blog/2013/08/27/rails-autoloading-hell)
