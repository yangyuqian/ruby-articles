# 引言

“千里之行，始于足下”, Ruby 内核类加载机制是一切类加载的根本，实际开发很少会直接接触这一块，然管中窥豹，知其所以然还是很有必要的.

# Ruby Kernel中的类加载

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
