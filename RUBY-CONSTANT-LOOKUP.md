# 引言

在 Ruby 中访问一个常量 A 时，最终是怎么找到的呢? 来看几个例子:

例1:

```
module A
  CREF = "*** CREF in module A"
end 

module A
  module B
    puts CREF
  end
end
```

例2:

```
module A
  CREF = "*** CREF in module A"
end

module A::B
  puts CREF
end
```

例3:

```
class A
  CREF = "*** CREF in module A"
end

B = Class.new(A) do
  puts CREF
end
```

例4:

```
class A
  CREF = "*** CREF in module A"
end

class B < A
  class << self
    puts CREF
  end
end
```

以上 4 个例子基本涵盖了本文将要介绍的解释器中的“常量查找”(Constant Lookup)机制, 这 4 个例子中哪些是可以正确执行的呢? 如果报错, 具体会报什么错?

这里先不揭晓答案, 读者可以自行在 irb console 中去执行这几段例程, 看看结果是怎么样的.


# Ruby 常量查找

很多开发语言中，“常量”的行为很简单，而 Ruby 中的“常量查找”机制给这个已经司空见惯的概念披上了神秘的面纱.

## nesting

nesting 是解释器用来编排(compose) Class/Module 上下层级关系的, 以下的定义方式中 B 获得的 nesting 就是自身以及上层的 namespace（不含 Object，这是一个全局回溯的点），即:

```
module A
  module B
    Module.nesting == [A::B, A]
  end
end
```

可以将 nesting 看成一颗多叉树，具体怎么编排树中的节点和具体写法有关，以下的定义方式中 B 获得的 nesting 仅包含自身:

```
module A::B
  Module.nesting == [A::B]
end
```

nesting 是解释器维护的一个 Class/Module 私有栈, 这个栈的读写有很多限制. 如传入的 block 不能读取这个栈（得到一个空数组）:

```
class A
end

B = Class.new(A) do
  def self.hi
    Class.nesting
  end
end

B.hi

class C < A
  puts Class.nesting.join(', ')

  def self.hi
    yield
  end
end

C.hi { Class.nesting }
```

有意思的，用匿名类的方式定义的 singleton 方法读到的 nesting 和直接在 class 里面定义的结果不同, 这个匿名类会先被 push 到 nesting 中，然后又 pop 出来:

```
class A
  class << self
    Class.nesting
  end
end
# -> [#<Class:A>, A]
```

总之, nesting 和代码的具体写法有关, 而且对 block 存在读取限制.

## 常量查找算法

内核中的常量查找算法的伪代码如下:

```
if 从 Module.nesting 找到常量定义？
  很开心的退出，返回常量引用
else
  while 遍历到整个结构树的顶层节点？
    if 从 Module.nesting.first.ancestor 找到常量定义?
      很开心的退出，返回常量引用
    else
      继续往 Module.nesting.first.ancestor 查找
    end
  end
end

if 上面的方式找不到
  去 Object 里面查找常量定义，这个时候再找不到就直接抛 NameError 了.
end
```



# 参考文献

[Everything you ever wanted to know about constant lookup in Ruby](http://cirw.in/blog/constant-lookup.html)

[Ruby constant lookup: The good, the bad and the ugly](http://makandracards.com/makandra/20633-ruby-constant-lookup-the-good-the-bad-and-the-ugly)

[Autoloading and Reloading Constants](http://guides.rubyonrails.org/autoloading_and_reloading_constants.html)

