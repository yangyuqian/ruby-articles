## 前言

了解一点类加载机制可以提升开发人员对于项目类结构的控制力，灵活安排一个app的gem

## Ruby Kernel中的类加载

### load(filename, wrap=false) → true/false

```
# 加载calendar.rb，并用匿名module包装文件内容，保护global namespace
load './calendar.rb', true
# 加载calendar.rb到global namespace
load './calendar.rb’
```

性质：

1. 每次都会加载，因此分布在多个文件中的类能够正确加载

2. 默认会从$LOAD_PATH查找文件($:)， $:.unshift ‘.'将当前目录加入$LOAD_PATH，然后load ‘calendar.rb’，这里必须要带rb后缀


### autoload(module, filename) → nil

```
autoload :Calendar, './calendar.rb’
```

性质：

1. 第一次访问Calendar的时候会尝试加载calendar.rb文件，然后查找内容中Calendar的类定义，如果没找到（可能该文件中没有）会报错

2. 调用autoload仅仅创建了一个钩子，并未真正加载


### require(name) → true or false

```
require './calendar.rb’
```

性质：

1. 默认会从$LOAD_PATH查找文件($:)，$:.unshift ‘.'，将当前目录加入$LOAD_PATH，然后load ‘calendar’，rb后缀不是必须的（区别于load方法），自动查找的后缀可以是so, o, dll

2. 抛开后缀的扩展，load每次都会加载，但require只加载一次，一旦某个文件被加载过一次，之后就不再加载，不管是基于$LOAD_PATH的相对路径还是绝对路径，只要是同一个文件，就只加载一次


### require_relative(string) → true or false

```
require_relative 'calendar'
```

性质：

1. 直接查找相对路径

2. 其他行为和require一致，每个文件也只加载一次



## 基于bundler的gem管理机制(以rails app为例)

首先，app启动的时候会默认执行 $APP_ROOT/config/boot.rb，根据Gemfile初始化$LOAD_PATH，将已经定义好的gem的lib目录加入$LOAD_PATH:

```
require 'rubygems'
# Set up gems listed in the Gemfile.
ENV['BUNDLE_GEMFILE'] ||= File.expand_path('../../Gemfile', __FILE__)
require 'bundler/setup' if File.exists?(ENV['BUNDLE_GEMFILE'])
```

然后，rails会执行 app/config/application.rb

```
require File.expand_path('../boot', __FILE__)
require 'rails/all'
Bundler.require(*Rails.groups(:assets => %w(development test)))
# 此处省略module定义，内容和具体的proj相关
```

其中Bundler初始化gem group的机制，和具体的RAILS_ENV相关(参见官方文档)，以下代码可以将自定义的demo group下的gems加入LOAD_PATH:

```
require 'bundler'
Bundler.require(:default ,:demo)  # 这里绕过了Rails.groups，RAILS_ENV就不影响具体的group加载了
```

实战：构造一个demo app，采用bundler来加载gem

[一个基于Bundler的类加载实例](https://github.com/yangyuqian/ruby-articles/blob/master/samples/demo_bundler.zip)

注：执行前请先bundle install


## ActiveSupport中类加载机制

### autoload

实例分析：

```
# add app root into LOAD_PATH
$:.unshift File.expand_path('../../',__FILE__)
module Demo
  # load user module through active_support/dependencies/auto_load
  # 在当前$LOAD_PATH下查找demo/user.rb，这里的demo对应了这一层的module名称
  extend ActiveSupport::Autoload
  autoload :User
end
```

注：这个例子稍微有些复杂（建议先了解前面的bundler机制），在rails中加载一个gem也可以这么干，区别在于gem中会在demo下弄一个demo.rb来作为整个gem的入口，而这里弄的是一个bin/demo，区别仅仅是$LOAD_PATH设置，机制是一致的

1. ActiveSupport::Autoload支持原声的autoload特性，并为其加入了默认值

2. autoload是为了解决类比较多的情况下存在复杂依赖时自动管理类加载的方案（A依赖B，加载A之前需要加载B）

3. 在某个类存在依赖的类时，也能自动加载

### eager_autoload

并非一种独立的类加载机制，而是基于autoload之上的一种线程安全的实现，在Rails 4中，这就显得不那么关键了，参见[Eager loading for greater good](http://blog.plataformatec.com.br/2012/08/eager-loading-for-greater-good)。

```
module Demo
  # load user module through active_support/dependencies/auto_load
  extend ActiveSupport::Autoload
  autoload :User
  # 这里采用eager_autoload记载所有的类
  eager_autoload do
    autoload :Role
    autoload :RoleDeco
  end
end
```

实例：[用ActiveSupport中的类加载来构造proj](https://github.com/yangyuqian/ruby-articles/blob/master/samples/demo_activesupport_autoload.zip)


## 参考文献：

[The Rails Initialization Process](http://guides.rubyonrails.org/initialization.html)

[How and Why Bundler Groups](http://yehudakatz.com/2010/05/09/the-how-and-why-of-bundler-groups/)

[Active Support 核心扩展](http://guides.ruby-china.org/active_support_core_extensions.html)

[Eager loading for greater good](http://blog.plataformatec.com.br/2012/08/eager-loading-for-greater-good)

[Gem Packaging: Best Practices](http://weblog.rubyonrails.org/2009/9/1/gem-packaging-best-practices/)

[Ways to load code](https://practicingruby.com/articles/ways-to-load-code)

[Rails启动过程](https://ruby-china.org/topics/21294)
