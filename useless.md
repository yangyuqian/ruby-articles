Ruby 中 Class/Module 都是以常量（Constant）的方式进行管理的. 而在解释器中 Module/Class 之间的嵌套关系可以简单的看作一个比较大的 Hash, 里面维护了顶层 Object 类到下面的所有 Module/Class 直接的引用关系，如：

```
module Auth
  class User
  end
end
```
可以抽象成以下的 Hash 表:

```
{
  'Object': {
    'Auth::User': User
  }
}
```

这让我们在代码的其他地方访问 User 类成为可能:

```
module Auth
  module Role
    User
  end
end
```

内核已经提供了查找类常量的机制(在后面的章节会提到). 在类结构比较复杂时，怎么去保证一个正确的 User 类在被使用到的时候已经加载了呢? 小项目中可以人工维护这个依赖加载, 在项目中如果还让开发人员去 “人肉” 维护整个类依赖树, 这将是个伪命题.

这时有人想到了一个突破口: 重写 Ruby 的 const_missing 方法:

```
module A
  def self.const_missing(name)
    # Find name, and load name
  end

  B
end
```

如果 A 用到 B 的时候它还没有被加载, 解释器发现找不到 B 就跑去告诉 const_missing 方法，说: “Find it, and load it”. 

然后? 是不是就没有然后了？因为似乎问题已经解决了: const_missing 高兴的加载了类（先别管怎么加载的）, 然后解释器就开心用这个类去做一些羞羞的事情.

当然不是！！！首先从内核类加载的角度来看, 不论怎么去加载一个类，一定有一个潜在的条件: 文件的位置已知.

试想如果能自动加载，而我们却需要为自动加载“人肉”维护一个大的 Hash 表, 告诉解释器应该加载哪个文件, 结果和直接用内核类加载就没本质区别了, 又绕回来了！
