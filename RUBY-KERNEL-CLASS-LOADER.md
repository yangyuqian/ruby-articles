## Ruby Kernel中的类加载

### Kernel.load(filename, wrap=false) → true/false

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


## 基于 Bundler 的 Gem 管理机制

### 启动 Rails App 时加载 Gem 的原理

Rails App 启动时会执行 $APP_ROOT/config/boot.rb:
  * 根据 Gemfile 初始化 $LOAD_PATH , 并将 Gemfile 中声明的 gem 的 lib 目录加入 $LOAD_PATH

以下是一个 boot.rb 的实际例子：

```
require 'rubygems'
# Set up gems listed in the Gemfile.
ENV['BUNDLE_GEMFILE'] ||= File.expand_path('../../Gemfile', __FILE__)
require 'bundler/setup' if File.exists?(ENV['BUNDLE_GEMFILE'])
```

Rails App 会执行 app/config/application.rb:
  * 根据 RAILS_ENV 将 Gemfile 中符合要求的 Gem group 加入 $LOAD_PATH

```
require File.expand_path('../boot', __FILE__)
require 'rails/all'
Bundler.require(*Rails.groups(:assets => %w(development test)))
# 此处省略module定义，内容和具体的项目相关
```



## 参考文献：


