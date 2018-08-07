---
layout: post
title:  "在ZJNT的一些对话和思考"
date:   2018-08-07 12:00:00 +0800
categories: 浅谈感想
---

今日只想写写英文。If you can understand Chinese only, sorry.

"ZhiJiang New Town is such a wonderful place to work in! It must great feeling that I can set my own company here some day.", I sighed. I bored of my live, I want to have a chance to change something, or even to change the world?
So what have we talked about?

### My first coding Python.
I was asked how I first coding Python. Actually, I was an iOS developer before then, I tried to write mobile app to express my ideas, but it needed a server-side, so I tried to use Python Django application as my backend. The Python Django is wonderful, its core mode is MTV(model/template/view), I can simply modeling my business in the model layer, and provides RESTful API to the client-side, it's the job of view. I was asked how do the server-side and client-side communicate, I answered that Python Django often to be use as web server, and I can use JSON serializer as template layer, so we can accept request and return response in JSON easily.
I was also asked how to manage sessions, I answered that at the beginning of a small project, we could use the default session manager of Django, it could store the sessions data in relational database or in-memory database. But once the demand that default session manager cannot satisfy, we have to think about using customize session manager. In addition, to get the session information is easy, all we need to do is to get it from the request object.

### About skynet?
We talk about skynet. I said that skynet is a service based tool, which had been trying hard to imitate the mode of erlang, was targeting to build a easier-erlang, I guessed Cloudwu wanted that. Because skynet use lua as its default language, such a simple script language. I also tried to explain the inner logic of skynet. Above all, erlang process is a core concept, it provides a system-like process in a erlang application, each erlang process is independent, we cannot share memory or objects between erlang processes straightly, but we can use something like pipeline to send message between erlang process. The implementation of skynet is similar, skynet use a lua virtual machine to simulate a erlang process, which Cloudwu defines that it calls skynet service. Each skynet service is also independent, unless we customize the skynet, or we can only use its call function or its send function to communication.
I was asked how the skynet works, I answered that each skynet application has a core dispatch queue, all the messages to dispatch will push to this dispatch queue safely. A skynet application has one thread to do the queue dispatch job, and one thread to do the timer job, and working threads that loop all the lua virtual machines, quantity of working thread use to be the same as the number of the CPU thread.
By the way, I don't know why Cloudwu give up the golang version of skynet, but using lua to do such an implementation is also a good ideas, lua virtual machine is light-weight enough. Co-routine is a clever way, we don't have to care about threads locking or manage system processes. I still tangled about the answer of Python GIL problem and too many system processes problem.
Is the skynet good enough? As the Cloudwu said, it's really not a tool that we can use straightly, a plenty of coding job we have to do on it. And I think the most important thing skynet has to improve is that the support for cluster. Skynet does not have a good service discovery middle-ware.

### What is the implementation of lua table?
When we were talking about lua, I complained about the implement of its table, it was awful, it mixed the concepts between array and dictionary. When you use it as a array, it can be a array; when you use it as a dictionary, it can be a dictionary; when you use it as a dictionary and a array, it don't refuse you! Lua table is like a black hole of everything, you can throw anything to it, but it's hard to know what you have throw to it.
What even worse is that, lua table does not give coder complete support, when you use a table as a array, actually it's still a dictionary inside.(P.S. I found I was wrong then. Actually, after reading the source code of lua, tables keep its elements in two parts: an array part and a hash part Non-negative integer keys are all candidates to be kept in the array part. I am so sorry that my comprehension was wrong.). Coder cannot remove an element of a array straightly(P.S. In a for loop), because it's not safety to do so, be careful when you have to delete a range of elements.
I was asked that how table implemented, I answered that table has a hashed dictionary inside, it stores everything even the array elements. And I found that I was wrong, those array elements store in array part of table. And will table change its store method determind to the content length, I think they tried to check me whether I knew the storage stragy or no. But actually lua won't change its storage method of table, regardless how long or how short the table content is. But it will resize the array part or the hash part determind by current size.

### How to generate a random number?
I was asked how to generate a random number, which number is between one to ten. But I haven't written python code frequently for almost one and a half years, what I still rememebered was that I should import a library named "random", so I tried to answer this question in lua, although they doubt that have I really ever written Python, but I've really written python for three years. It's confuse to remember the detail of Python, the lua coding experience scour my Python coding memory. Firstly, we have to use math library, call the randomseed method to refresh the random seed of the lua virtual machine. Secondly, we can call the random method straigthly, to get a random float, which is between zero and one. Lastly, multiply this random float by ten, then divisible one.
They were interested about why should the random seed to be refreshed. I tried to explain how our computers generate a random number, actually we mostly use the clock of CPU to generate a random number, and we refresh random seed in lua to prevent we get the same random seed. I was asked that can we generate the same random numbers in different machines, I was afraid not, even though we use the same CPU architecture, and in the same time, but I was told that as long as the random seed is the same, the random number would be the same. I was confused.
After the conversation, I tried to use the same random seed to generate random numbers, but I still got different numbers, maybe I've understand mistaken somewhere. And I looked up the document of Python, generate a random number in Python is even simpler.
```
import random
random.randint(1, 10)
```
I was smilence(smile and be silence). I determine to pick up my losed Python skill.

### How to hot reload a python module?
It's a common scene that we need to hot reload the application code. I was asked how to do hot reload in Python. Since I was long time not coding Python, I had the only impression was that we could remove the reference of a module in a Python file, after that, import the module into Python again. After I answered, they told me that Python has sys.module, we could delete the module in sys.module by name to unload a module, then we could do the reload job.
And we also talked about the hot reload in skynet, we used to have two way, to restart a service in shell, or to clear cache of a service.

### Frame synchronization and gaming room design
They asked me about the frame synchronization, I knew they wanted to ask me the server-side, but my answer was about the client-side frame synchronization. I shared my experience about how the iOS client is working, it's driven by a event loop, it's an endless loop until the loop is breaked, and it has a timer that makes each loop not faster than sixty frames per second. Events such as screen touches will be capture and pass to the current loop.

### Database indexing

### How to change a wrong user data

To be continued
