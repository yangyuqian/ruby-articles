# $LOAD_PATH

在一个 Rails 项目中, 有很多的第三方类库（Gem）, 还有项目自身的文件, App 如何管理这些类库？

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

当然，即便有 $LOAD_PATH 还是不够的, 很多第三方类库里面的依赖使用者显然是不清楚的, 要让程序员写出好代码，又要维护本不是自己开发的一些第三方类库的依赖, 那我还是回去写汇编好了.

Ruby 的世界不能容忍每个人都去手动维护一坨 $LOAD_PATH 初始化脚本，那么就有了 Bundler, 提供了一系列优雅地管理第三方类库 $LOAD_PATH 的解决方案，详情参见 [基于 Bundler 的 Gem 管理机制](https://github.com/yangyuqian/ruby-articles/blob/master/BUNDLER.md)

# Ruby Kernel 中的类加载

Ruby 内核提供了 4 个类加载命令，分别是 load, autoload, require, require_relative, 分别对应了不同的使用场景，可谓做到了“小的可以打蚊子，大的可以打飞机”.

## Kernel.load(filename, wrap=false) → true/false

load 命令提供了一种最原始的方法，即每次都会重新加载整个文件，刷新内存中的类定义.

新建一个calendar.rb, 内容如下:

```
# calendar.rb
class Calendar
  def initialize(month, year)
    @month = month
    @year  = year
  end

  # A simple wrapper around the *nix cal command.
  def to_s
    IO.popen(["cal", @month.to_s, @year.to_s]) { |io| io.read }
  end
end

puts Calendar.new(8, 2011)
```

开一个 irb console 直接加载 calendar.rb:

```
# 这里可以给绝对路径，也可以是相对路径
irb(main):001:0> load './calendar.rb'
    August 2011
Su Mo Tu We Th Fr Sa
    1  2  3  4  5  6
 7  8  9 10 11 12 13
14 15 16 17 18 19 20
21 22 23 24 25 26 27
28 29 30 31

=> true
```

这存在一个问题，即 calendar.rb 加载进来后可能会影响当前 namespace 的一些状态, 也可能被当前 namespace 的状态影响，比如常量的值等等. 如果希望 calendar.rb的内容 悄悄的加载, 不影响当前 namespace 中的状态，load 命令支持用一个匿名 Module 包装被加载的内容，从而保证了这个文件里面的东西都是在限定范围内执行:

```
irb(main):001:0> load './calendar.rb', true
    August 2011
Su Mo Tu We Th Fr Sa
    1  2  3  4  5  6
 7  8  9 10 11 12 13
14 15 16 17 18 19 20
21 22 23 24 25 26 27
28 29 30 31

=> true
```

觉得相对/绝对路径太麻烦？可以将当前路径加入 $LOAD_PATH:

```
irb(main):001:0> $:.unshift File.dirname(__FILE__)
```

然后加载文件只需要给出文件名:

```
irb(main):002:0> load 'calendar.rb'
    August 2011
Su Mo Tu We Th Fr Sa
    1  2  3  4  5  6
 7  8  9 10 11 12 13
14 15 16 17 18 19 20
21 22 23 24 25 26 27
28 29 30 31

=> true
```

### Kernel.autoload(module, filename) → nil

load 命令每次都加载类有些浪费，很多类并不是一开始就需要，可以用 autoload 来先创建一个钩子，等到真的访问到的时候再加载：

```
autoload :Calendar, './calendar.rb'
```

但这种方式有个问题: 相同常量如果多次定义 autoload 钩子，只有最后一个会被触发. 设想在实际开发中，类定义可能分布在多个文件中，所以这种方式并不常用.

### Kernel.require(name) → true or false

和 autoload 一样, require 想解决的也是性能问题: require 只在第一次被调用的时候被触发，之后针对相同文件的 require 就不会真正执行了:

```
irb(main):001:0> require './calendar.rb'
    August 2011
Su Mo Tu We Th Fr Sa
    1  2  3  4  5  6
 7  8  9 10 11 12 13
14 15 16 17 18 19 20
21 22 23 24 25 26 27
28 29 30 31

=> true
irb(main):002:0> require './calendar.rb'
=> false
```

require 比 load 更强大一些, load 是必须给出文件后缀的，而 require 可以不给出后缀，且相同的名字对 .so .o .dll都是有效的：

```
irb(main):001:0> require './calendar.rb'
    August 2011
Su Mo Tu We Th Fr Sa
    1  2  3  4  5  6
 7  8  9 10 11 12 13
14 15 16 17 18 19 20
21 22 23 24 25 26 27
28 29 30 31

=> true
irb(main):002:0> require './calendar'
=> false
```

require 和 load 也都会读取 $LOAD_PATH，因此如果将当前目录加入 $LOAD_PATH，require 也就可以不给相对路径，只给一个文件名了：

```
irb(main):001:0> $:.unshift File.dirname(__FILE__)
=> [".", "/Library/Ruby/Site/2.0.0", "/Library/Ruby/Site/2.0.0/x86_64-darwin14", "/Library/Ruby/Site/2.0.0/universal-darwin14", "/Library/Ruby/Site", "/System/Library/Frameworks/Ruby.framework/Versions/2.0/usr/lib/ruby/vendor_ruby/2.0.0", "/System/Library/Frameworks/Ruby.framework/Versions/2.0/usr/lib/ruby/vendor_ruby/2.0.0/x86_64-darwin14", "/System/Library/Frameworks/Ruby.framework/Versions/2.0/usr/lib/ruby/vendor_ruby/2.0.0/universal-darwin14", "/System/Library/Frameworks/Ruby.framework/Versions/2.0/usr/lib/ruby/vendor_ruby", "/System/Library/Frameworks/Ruby.framework/Versions/2.0/usr/lib/ruby/2.0.0", "/System/Library/Frameworks/Ruby.framework/Versions/2.0/usr/lib/ruby/2.0.0/x86_64-darwin14", "/System/Library/Frameworks/Ruby.framework/Versions/2.0/usr/lib/ruby/2.0.0/universal-darwin14"]
irb(main):002:0> require 'calendar'
    August 2011
Su Mo Tu We Th Fr Sa
    1  2  3  4  5  6
 7  8  9 10 11 12 13
14 15 16 17 18 19 20
21 22 23 24 25 26 27
28 29 30 31

=> true
```

### Kernel.require_relative(string) → true or false

require_relative 相当于是默认将当前路径加入了 $LOAD_PATH，不用给相对路径或绝对路径, 其他和 require 是一致的：

```
irb(main):001:0> require_relative 'calendar'
    August 2011
Su Mo Tu We Th Fr Sa
    1  2  3  4  5  6
 7  8  9 10 11 12 13
14 15 16 17 18 19 20
21 22 23 24 25 26 27
28 29 30 31

=> true
irb(main):002:0> require_relative 'calendar.rb'
=> false
```


# 参考文献

[Ways to load code](https://practicingruby.com/articles/ways-to-load-code)
