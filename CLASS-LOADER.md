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

### 实例

构造一个 Demo App, 采用 Bundler 来加载 Gem:

[一个基于Bundler的类加载实例](https://github.com/yangyuqian/ruby-articles/blob/master/samples/demo_bundler.zip)

注：执行前请先bundle install


## ActiveSupport中类加载机制

### autoload

创建以下目录结构：
  * $APP_ROOT/demo.rb, 定义了类加载的入口
    * $APP_ROOT/demo/user.rb, 定义了一个 User module

$APP_ROOT/demo.rb 内容如下:

```
# 此处省略了 Bundler 管理 active_support 依赖的内容，见前面章节

# Add the app root into LOAD_PATH
$:.unshift File.expand_path('../../',__FILE__)

module Demo
  # load user module through active_support/dependencies/auto_load
  # 在当前 $LOAD_PATH 下查找 demo/user.rb, 这里的 demo 对应了当前 module
  extend ActiveSupport::Autoload
  autoload :User
end
```

Tips:

1. Rails 中根据 $LOAD_PATH 来管理了很多默认的查找路径
  * 如 User , 如不声明，就会在当前的 module/class 的相对路径下去查找

2. 在 Rails 3 及以下版本中是线程不安全的

### eager_autoload

并非一种独立的类加载机制，而是基于 autoload 之上的一种线程安全的实现，在 Rails 4 中，这就显得不那么关键了，参见 [Eager loading for greater good](http://blog.plataformatec.com.br/2012/08/eager-loading-for-greater-good)

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

### 实例

[用ActiveSupport中的类加载来构造proj](https://github.com/yangyuqian/ruby-articles/blob/master/samples/demo_activesupport_autoload.zip)


## 参考文献：

[The Rails Initialization Process](http://guides.rubyonrails.org/initialization.html)

[How and Why Bundler Groups](http://yehudakatz.com/2010/05/09/the-how-and-why-of-bundler-groups/)

[Active Support 核心扩展](http://guides.ruby-china.org/active_support_core_extensions.html)

[Eager loading for greater good](http://blog.plataformatec.com.br/2012/08/eager-loading-for-greater-good)

[Gem Packaging: Best Practices](http://weblog.rubyonrails.org/2009/9/1/gem-packaging-best-practices/)

[Ways to load code](https://practicingruby.com/articles/ways-to-load-code)

[Rails启动过程](https://ruby-china.org/topics/21294)
