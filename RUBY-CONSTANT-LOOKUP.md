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


# Ruby 内核提供的常量查找算法

## 



# 参考文献

[Everything you ever wanted to know about constant lookup in Ruby](http://cirw.in/blog/constant-lookup.html)

[Ruby constant lookup: The good, the bad and the ugly](http://makandracards.com/makandra/20633-ruby-constant-lookup-the-good-the-bad-and-the-ugly)

[Autoloading and Reloading Constants](http://guides.rubyonrails.org/autoloading_and_reloading_constants.html)