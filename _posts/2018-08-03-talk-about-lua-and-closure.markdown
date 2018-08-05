---
layout: post
title:  "聊聊lua和closure"
date:   2018-08-03 22:00:00 +0800
categories: 浅谈
---

# 聊聊lua和closure

### 聊聊Lua

Lua是一个好语言，我们公司目前的项目都大量使用Lua，用其编写业务代码——服务器端方面是使用云风的skynet，开发语言自然用的就是Lua；客户端方面则是使用C#项目集成LuaVM，把Lua代码当作成资源包来更新加载，实现客户端的热更新，客户端的同学再也不需要担心CSharp代码出问题要重新发版本审核了。

有一天吃完晚饭和同事散步聊天，刚好聊起了Lua。其实我在进来目前公司之前，对Lua这个语言不甚了解，甚至刚开始的时候对其也没有太多的好感，但这一切在我写了数月的Lua代码之后尝试写写Python代码时改变了，这时我的大神同事笑说，他也觉得有类似的感受：写了Lua之后就写不回Python了，写了Python也很难写回Lua。例如，在写Lua时习惯了直接访问table对象中的属性，如果内容不存在只会返回nil，但回到Python，直接访问一个不存在的属性会raise AttributeError。

然后又和同事聊了不少Lua的问题，我和我的同事都认为：

1. 应该把table至少要把list和dictionary的概念分开

我们都可以理解Lua要做一个十分轻量级的VM，所以不会提供很多语法支持，但把List和Dictionary这两个概念混在table这种数据结构里面，这种做法到底好不好？至少我觉得是不好的，甚至是荒唐。List和Dictionary根本上就是两种完全不同的数据结构，而Lua的table实际上是一个Dictionary，那table就发挥好它作为Dictionary的角色就好了，Lua应该有一个独立的List的数据结构。但Lua的作者偏偏不要，他就是要用数字Key值下标的方式来实现List，我们可以把table当成List和Dictionary，也可以同时把List和Dictionary添加到table中。不可理喻。

2. table作为List时的用法十分奇葩

我们可以用table.insert函数来给table添加List元素，只要你只做insert操作和get操作，一切都是正常的。但当你要进行table的修改操作时，一切就变得悲剧了，你可以remove一个元素，但如果你没有研究过如何安全地移除一个元素，那么这个table就很有可能出错了，例如虽然你remove了一个元素，但在sharp一个table的长度的时候，table的长度还是remove之前的长度呀，这算是哪门子的支持？顺带一提，sharp操作只对table中的List部分生效，它并不把Dictionary部分的算在内，这也是很令人费解的地方。

3. table的下标竟然是从1开始？

这个……作者的意思是要让编程新手入门容易理解。好像Pascal和Matlab之类的语言也是这么处理，虽然我觉得这个十分值得吐槽，但算了，Coder习惯就好……

### 聊聊closure

我的同事为了解决Lua的这个问题，动手为客户端的项目封装过一个List的对象，我问他他是如何做到保护table内部数据的？Lua连像Python一样的下划线开头的变量名解释成私有的功能都没有提供。他给出的方案却令我吃惊：

```
new_list = function()
    local obj = {}
    local list = {}
    local list_count = 0

    obj.add = function(element)
        obj.insert(list, element)
        list_count = list_count + 1
    end
    obj.count = function()
        return list_count
    end
    ......

    return obj
end
```

我当时看了这代码十分疑惑，甚至是惊奇——难道这就不会引起悬空引用吗？当new_list()的整个函数代码块执行完之后，obj的引用被返回了，但list和list_count的引用应该被释放了呀，add和count函数是如何调用这两个“悬空”的引用的？
我的同事说：“这是闭包呀！”
我认真地想了一想，才恍然大悟，原来还有这么猥琐的方法来实现Lua的变量隐藏。表面上看，当这个new_list函数return的时候，这两个local方法已经是悬空甚至是已经销毁了的，但实际上因为add和count函数中引用了list和list_count变量这个上下文，使得list和list_count变量是有被引用的，因此确保了其在GC的时候不会被回收。其效果则是，当调用new_list函数返回的对象中的add和count方法，的确是可以访问到具体的变量，但是又因为没有办法可以拿到这两个变量的引用，因此可以视为这两个变量时私有的安全的。这方法真的是绝了。
当我想到这里，不禁想起了lisp一下，这种写代码的思路和方式，不就是lisp吗？
