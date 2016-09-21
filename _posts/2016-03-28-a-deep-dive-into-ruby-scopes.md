---
layout: post
title: "A Deep Dive into Ruby Scopes"
date: 2016-03-28
categories: 翻译
---
##### 感谢作者 Thijs Cadier [原文链接](http://blog.codeship.com/a-deep-dive-into-ruby-scopes/?utm_source=rubyweekly&utm_medium=email)


Ruby 是纯碎以面向对象的方式来设计的。在 Ruby 中，一切皆对象。

面向对象的设计为属性（properties）和动作 （actions）提供了封装。封装的目的在于保护方法和数据
免受外部的干扰和滥用。有了封装，所有的东西都只能在特定的作用域内才可以被使用。Ruby 中有几种不同
的作用域，它们分别是全局作用域（global），实例作用域（instance），和局部作用域（local）。这些
是 Ruby 中主要的作用域，但是仍有一些诸如类变量和使用的词法作用域（lexical scope）的
refinements 这样的例外存在。

理解 Ruby 的作用域将有助于你提升对 Ruby 的使用。我完成了一个有深度的概述来向你展示它们能如
何协助你写出更优美的代码。

### 封装

让我们一个被封装的局部变量的例子开始。（你使用等号赋值时比如 `a = 1` 时，就创建了局部变量）。
首先，我将向你展示一个没有被封装且写于 begin-end 块中的代码以及一个简单的方法定义。

    begin
      a = 4
    end
    puts a if defined?(a)
    # 4

    def local_var_example
      b = 4
    end
    local_var_example
    puts b if defined?(b)
    # => nil

这里我们能够很轻易的发现超出方法的作用域后被我们赋值为 4 的局部变量 `b` 不存在。如此，你就能在
方法中尽情的写代码而丝毫不用担心变量们会泄露出来。局部变量的作用域范围非常有限；你不能在定义
方法之前，先写的局部变量不能够被获取。

    a = "example"
    def a?
      puts a
    end

    a?
    #NameError: undefined local variable or method `a' for main:Object

If you want to draw in the environment and access local scope from outside a method definition, you may use closure definitions such as define_method, proc, or lambda.
如果你希望引入环境变量，并能从外部访问方法定义中的局部作用域，你可以使用像是 `define_method`，`proc`
，或是 `lambda` 这类闭包。

    word = "moo"

    define_method :x do
      puts word
    end
    x
    # moo

    y = proc {puts word}
    y.call
    # moo

    z = lambda {puts word}
    z.call
    # moo

### 警告

局部变量的优先级高于同名的方法。为了明确你想调用的是方法而非同名的局部变量时，你可以使用 `send`
方法。

    a = 4
    def a
      5
    end

    puts a
    # 4
    send :a
    # => 5

### 实例



### 单例

将它说成 "singleton instance" 对我而言有点重复，但是为了将它与单例模式以及 `singleton_class`
对象（大部分 Ruby 对象都，并且这不是它是对象的一个实例，但它自己的一个额外的单实例）加以区分。

Ruby 被设计成一切都是 Object 对象的一个实例，因此会有一个单例。这意味着其本身也拥有一个独立的
 self 存在。是的，这初看起来令人困惑。但一旦你看到每一个模块，类和对象作为它们自己的单例可能会
 或可能不会从他们的定义中创造出更多的单例，然后事情就变得清晰。

### 全局变量

当你在顶层作用域写代码时，你就是在全局作用域写代码。其中的局部变量不会跟任何作用域交错，实例变量可以被
局部的方法访问，方法和类都全局有效。

    local_variable = 1     # not available in any other scope

    @instance_variable = 2 # available within methods in the same scope

    $global_variable = 3   # available everywhere

    CONSTANT = 4           # available everywhere

    def a_method           # available everywhere
    end

    class Klass            # available everywhere
    end

    module Mod             # available everywhere
    end

Now the way method definitions are managed in the global namespace is quite interesting. As you may recall that everything in Ruby is an object, they are all also instances of the Object class itself. So the way that global methods are handled is that they are defined as private instance methods on the Object class.
现在定义于全局命名空间中的方法非常有趣。你可以在 Ruby 中的任意对象上加以调用，它们也同样

### 绑定（Binding）：作用域中的例外

Binding object 是唯一能让你在作用域之外传递修改局部变量的对象。你只需要使用 `binding` 方法就
能创建一个 binding。它会给局部环境创造一个 binding。你可以将这个 binding 传入其他作用域内并
能访问到 binding 实例化时存在的局部变量。

    module A
      def self.a(bnd)
        printf "%s\n", bnd.local_variables

        x = bnd.local_variable_get :x
        y = bnd.local_variable_get :y
        z = bnd.local_variable_get :z

        printf "%s\n", [x, y, z]

        bnd.local_variable_set(:z, x + y)
      end
    end

    module B
      def self.b
        x = 1
        y = 7
        z = 0

        A.a(binding)

        puts "x + y = #{z}"
      end
    end

    B.b
    # [:x, :y, :z]
    # [1, 7, 0]
    # x + y = 8

局部变量 `z` 的值在一个不同的作用域 `A` 中被改变，其结果在作用域 B 中显示了出来。
### 类变量

因为类变量会影响到同一个类的所有的实例，所有很少被使用。如果你更改了一个实例中的类变量，那其他所
有的实例变量中的类变量都会随之改变。

    class Building
      def initialize
        @@state ||= :built
      end

      def state(value = nil)
        @@state = value if value
        @@state
      end
    end

    library = Building.new
    office = Building.new

    library.state
    # => :built
    office.state
    # => :built

    office.state :demolished
    # => :demolished
    library.state
    # => :demolished

如你所见，如果你没有管理好类变量，那它可能会导致奇异的值出现。明智的做法是考虑将类变量放置只读属性的值或是放
入一线程安全的系统中。

### 总结

Ruby 是一门被设计用来使程序员快乐的语言，而理解它的作用域能使你更全面的利用这门语言。拥有它，你
就可以在设计阶段采用很多策略从而有助于你拥有一个更优美的代码库。
我推荐学习好的面向对象设计。每个语言/特性仅仅是个工具，而工具只有在理解和掌握的前提下才能发挥出
最大效率。作为 Ruby 的核心设计之一的封装，如果你按照它所设计的方式使用，那它将会很好的服务于你。
归功于 Ruby 是一门如此灵活的语言，我们有很大的自由度来使用它。
