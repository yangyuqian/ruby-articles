# 引言

Bundler 提供了复杂的第三方 Gem 与项目代码的 $LOAD_PATH 自动化管理的解决方案.

# 基于 Bundler 的 Gem 管理机制

## 启动 Rails App 时加载 Gem 的原理

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

# 参考文献

[The Rails Initialization Process](http://guides.rubyonrails.org/initialization.html)

[How and Why Bundler Groups](http://yehudakatz.com/2010/05/09/the-how-and-why-of-bundler-groups/)

[How and Why Bundler Groups](http://yehudakatz.com/2010/05/09/the-how-and-why-of-bundler-groups/)

[Gem Packaging: Best Practices](http://weblog.rubyonrails.org/2009/9/1/gem-packaging-best-practices/)
