# 引言

“千里之行，始于足下”, 如果说 Ruby 内核类加载是肉体，Rails 中提供的“启发式”类加载就是灵魂，灵魂脱离了肉体就是纸上谈兵.

Ruby 内核类加载机制是一切类加载的根本，实际开发虽少有直接用到, 但做到知其所以然对于有志的技术青年还是很有必要的.

# $LOAD_PATH

不论什么样的开发语言, 避免不了首先要解决代码文件管理的问题. 换言之, 必须要把一个类在某个地方定义出来才能用.

每个项目都有很多的类，在一个 Rails 项目中, 可能还会有很多的第三方类库（Gem）, App 如何管理这些类库？

在项目中完成业务代码后，怎么告诉 Ruby 的解释器说，把某个类加载进来？

以上这两个问题是同质的，显然应该有一个相同的逻辑来处理它们.

如果要对第三方的类库和项目代码进行统一管理，又不是灵活性，就势必要对文件做一个抽象， 就好像有一个文件系统，下面挂了很多个类库，以及业务代码，加载类的时候，只需要去这些地方找就行了，这个文件系统的抽象就是本章要介绍的 $LOAD_PATH.

Ruby 内核并不要求统一管理类，理论上我们的类文件可以分布在系统的各个角落，意味着我们需要为每个文件制定一个绝对路径活着相对路径，想想都那么蛋疼！

首先，打开一个 irb console, 默认的 $LOAD_PATH， 实际上是一个 String Array:

```
2.1.5 :001 > $:
 => ["/home/vagrant/.rvm/rubies/ruby-2.1.5/lib/ruby/site_ruby/2.1.0", "/home/vagrant/.rvm/rubies/ruby-2.1.5/lib/ruby/site_ruby/2.1.0/x86_64-linux", "/home/vagrant/.rvm/rubies/ruby-2.1.5/lib/ruby/site_ruby", "/home/vagrant/.rvm/rubies/ruby-2.1.5/lib/ruby/vendor_ruby/2.1.0", "/home/vagrant/.rvm/rubies/ruby-2.1.5/lib/ruby/vendor_ruby/2.1.0/x86_64-linux", "/home/vagrant/.rvm/rubies/ruby-2.1.5/lib/ruby/vendor_ruby", "/home/vagrant/.rvm/rubies/ruby-2.1.5/lib/ruby/2.1.0", "/home/vagrant/.rvm/rubies/ruby-2.1.5/lib/ruby/2.1.0/x86_64-linux"] 
```

这样就可以很容易将新的类库对应的跟路径加入 $LOAD_PATH，以下代码会将当前的目录加入 $LOAD_PATH:

```
$:.unshift File.dirname(__FILE__)
```

# Ruby Kernel 中的类加载

Ruby 内核主要提供了 4 个类加载命令，分别是 load, autoload, require, require_relative.

## Kernel.load(filename, wrap=false) → true/false

加载 calendar.rb，并用匿名module包装文件内容，保护global namespace

```
load './calendar.rb', true
```

加载calendar.rb到global namespace

```
load './calendar.rb’
```

Tips:

1. 默认会从 $LOAD_PATH 查找文件

2. 每次调用都会加载文件

3. 在使用 rails console 调试过程中，如果修改了类的内容，可以采用 load 来加载最新的类定义


### Kernel.autoload(module, filename) → nil

```
autoload :Calendar, './calendar.rb’
```

Tips:

1. 调用 autoload 仅仅创建了一个钩子，并未真正加载

2. 如果访问某个类定义触发了一个 autoload 的钩子，就会去尝试先加载文件后读取类定义

3. 只会真正加载一次

### Kernel.require(name) → true or false

```
require './calendar.rb’
```

Tips:

1. 默认会从 $LOAD_PATH 查找文件

2. rb 后缀不是必须的（区别于load方法），自动查找的后缀可以是 so, o, dll

3. Kernel.load 每次都会加载，但 Kernel.require 对同一个文件只加载一次


### Kernel.require_relative(string) → true or false

```
require_relative 'calendar'
```

Tips:

1. 行为和 Kernel.require 一致，区别在于 Kernel.require_relative 会从将当前目录也加入 $LOAD_PATH






# 参考文献

[Ways to load code](https://practicingruby.com/articles/ways-to-load-code)
