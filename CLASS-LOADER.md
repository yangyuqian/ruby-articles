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

3. 如果同时出现了多次相同类的autoload（文件路径可能不同），只有最后一个会起作用，在同一个类分布在多个文件中时，类定义将不能完整加载


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

首先，app启动的时候会默认执行<APP_ROOT>/config/boot.rb，根据Gemfile初始化$LOAD_PATH，将已经定义好的gem的lib目录加入$LOAD_PATH:

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

接着，Bundler会加载一些group，和具体的RAILS_ENV相关(参见官方文档)，这样可以加载自定义的demo group下的gems:

```
require 'bundler'
Bundler.require(:default ,:demo)  # 这里绕过了Rails.groups，RAILS_ENV就不影响具体的group加载了，实际开发里面不会这样
```

通过这样的方式将定义了的gem全部加载到内存中，提供给业务代码

实战：构造一个demo app，采用bundler来加载gem

[一个基于Bundler的类加载实例](https://github.com/yangyuqian/ruby-articles/blob/master/samples/demo_bundler.zip)

注：执行前请先bundle install
