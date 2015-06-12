# 引言

Bundler 提供了复杂的第三方 Gem 与项目代码的 $LOAD_PATH 自动化管理的解决方案.

# 基于 Bundler 的 Gem 管理机制

Rails App 在 Gemfile 声明的 Gem 在启动的时候就会通过 Bundler 整理好并构建正确的 $LOAD_PATH, 大体分为 2 步：

1. 启动时会执行 $APP_ROOT/config/boot.rb:
  * 根据 Gemfile 初始化 $LOAD_PATH , 并将 Gemfile 中声明的 gem 的 lib 目录加入 $LOAD_PATH

```
# $APP_ROOT/config/boot.rb
require 'rubygems'
ENV['BUNDLE_GEMFILE'] ||= File.expand_path('../../Gemfile', __FILE__)
require 'bundler/setup' if File.exists?(ENV['BUNDLE_GEMFILE'])
```

2. 执行 app/config/application.rb
  * 根据 RAILS_ENV 将 Gemfile 中符合要求的 Gem group 加入 $LOAD_PATH

```
# app/config/application.rb
require File.expand_path('../boot', __FILE__)
require 'rails/all'
Bundler.require(*Rails.groups(:assets => %w(development test)))
# 此处省略module定义，内容和具体的项目相关
```

这样一个 Rails App 就知道自己在运行时的 $LOAD_PATH, 接下来就可以对类文件按需加载了.

# 参考文献

[The Rails Initialization Process](http://guides.rubyonrails.org/initialization.html)

[How and Why Bundler Groups](http://yehudakatz.com/2010/05/09/the-how-and-why-of-bundler-groups/)

[How and Why Bundler Groups](http://yehudakatz.com/2010/05/09/the-how-and-why-of-bundler-groups/)

[Gem Packaging: Best Practices](http://weblog.rubyonrails.org/2009/9/1/gem-packaging-best-practices/)
