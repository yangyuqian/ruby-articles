采用内核提供的类加载方式, 如果在运行时发现类没有定义

Ruby 解释器对定义方式不同的类给出的 “常量字典” 是不同的，例如：

采用嵌套方式定义的一个Auth::User的 “常量字典” 是: ['Auth::User', 'Auth']

```
# user.rb
module Auth
  class User
  end
end
```

和用简洁的写法定义的User类得到的 “常量字典” 是: ['Auth::User']

```
# user.rb 
module Auth::User
end
```

这带来了一个问题，