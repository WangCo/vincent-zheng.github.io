title: nodejs addon
---
## 前言
Addons是一个动态链接对象，可以为C/C++库提供一个链接的方法，为node.js提供更高的性能和多线程的能力。不过同时这个东西对于入门新手也是蛮复杂的~~ ，再加上中文文档偏少所以对于我这种英语学渣来说挺坑爹的~~ 。首先，它主要依赖了以下几个库：
- V8 JavaScript，这个是V8 引擎的库，相应的文档可以通过google chrome V8 可以找到~~
- libuv， 这个感觉是node.js的精髓，它是node.js的跨平台抽象层，用于抽象Windows的IOCP和Unix的libev，简单来讲就是异步处理机制的库。
- 其他一些dependencies，可以在deps目录下找到，
    - windows下是  C:\Users\UserName\.node-gyp\0.12.0\deps
    - linux下是 /usr/include/nodejs/deps


### Hello world 
>  这部分来自官档，可以点击[这里](https://nodejs.org/api/addons.html)查看

与c++代码等义的JS代码
```
module.exports.hello = function() { return 'world'; };
```

首先要在目录下创建一个 hello.cc文件，这个就是c++的源程序
```
// hello.cc
#include <node.h>

using namespace v8;

void Method(const FunctionCallbackInfo<Value>& args) {
  Isolate* isolate = Isolate::GetCurrent();
  HandleScope scope(isolate);
  args.GetReturnValue().Set(String::NewFromUtf8(isolate, "world"));
}

void init(Handle<Object> exports) {
  NODE_SET_METHOD(exports, "hello", Method);
}

NODE_MODULE(addon, init)
```

解释一下，这里的init函数是用于初始化输出项，相当于JS代码中的exports，exports是一个载入的文件作用于中的全局变量，而NODE_MODULE这里不是一个函数~~ ，这是一个宏定义，详情可以看node.h，
```
#define NODE_MODULE(modname, regfunc)                                 \
  NODE_MODULE_X(modname, regfunc, NULL, 0)
```

最后，再写上一个binding.gyp就可以了，binding.gyp大概相当于一个编译脚本的配置文件，用于生成相应平台下的编译脚本的，在windows下创建一个VS工程，而在Linux下面则会生成makefile等东西，编译后的生成文件会放于build/Release/目录下，后缀名为.node,不同平台下皆一样，这里有必要说明一下，后缀名虽然相同，但是不可以由于编译平台不一样所以不能直接复制到不同操作系统下使用~~ 。当然有时候也会遇到坑，导致编译成功了但是没有在build/Release目录下找到相应的文件，例如libxmljs这个库的某些版本~~ 。
binding.gyp： 
```
{
  "targets": [
    {
      "target_name": "addon",
      "sources": [ "hello.cc" ]
    }
  ]
}
```

编译命令：
> PS： 这里有时也会遇到坑，因为我更新了Ubuntu里面的node的版本，而编译依赖的源码没有被更新到，所以导致编译的时候会出错~~ ，
> 修复方法：
> sudo apt-get remove gyp 
> sudo npm remove -g node-gyp
> sudo apt-get install gyp
> sudo npm install -g node-gyp
> // 简单来讲就是重装这个东西 ~~ 

```
// 进入工程目录下
node-gyp configure
node-gyp build
```

最后在JS代码中调用这个addon：
> PS: 如果不是./让他选择本项目中寻找会报出无法找到模块的错误。 



```
// hello.js
var addon = require('./build/Release/addon');

console.log(addon.hello()); // 'world'
```

以上主要参考于官方文档，下面加点如何使用c++11的方法 ~~ ， 个人比较喜欢c++11，感觉到了这个版本c++写起来才比较有感 ~~

### node.js addon c++11 thread使用
> 本源码只在VS2013和Ubuntu下的gcc4.9.2编译测试过，其他平台没有测试过 ~~ ， 也不知道其他平台对c++11支持情况如何。

C++ 源码不是主要，主要是看binding.gyp如何编写，所以c++部分代码直接贴上。

C++ 源码： 

```  
    // hello.cxx
    #include <node.h>
    #include <iostream>
    #include <thread>
    
    #ifdef WIN
    #include <windows.h>
    #endif
    
    #ifdef UNIX
    #include <unistd.h>
    #endif
    
    using namespace std;
    using namespace v8;
    
    void func(const FunctionCallbackInfo<Value> & args, Isolate * isolate, int i) {
    #ifdef WIN
    	Sleep(3000);
    #endif
    #ifdef UNIX
    	sleep(3);
    #endif
    
    	if (i == 0) {
    		args.GetReturnValue().Set(String::NewFromUtf8(isolate, "hello vincent!!!"));
    	}
    	cout << "thread" << i <<endl;
    }
    
    void hello(const FunctionCallbackInfo<Value> & args) {
    	Isolate * isolate = Isolate::GetCurrent();
    	HandleScope scope(isolate);
    	thread t[2];
    	int i = 0;
    	t[0] = thread(func, args, isolate, i ++);
    	t[1] = thread(func, args, isolate, i);
    	t[0].join();
    	t[1].join();
    	cout<<"end"<<endl;
    }
    
    extern "C" void init(Handle<Object> exports) {
    	NODE_SET_METHOD(exports, "hello", hello);
    }
    
    NODE_MODULE(addon, init)
``` 

binding.gyp:
> 关键在于要定义那个cflags，这个地方没有在官档中找到，我是从addon.gypi里面摸索出来的。


```
{
	'targets': [
		{
			'include_dirs': [
		      '<(node_root_dir)/src',
		      '<(node_root_dir)/deps/uv/include',
		      '<(node_root_dir)/deps/v8/include'
		    ],
			'target_name': 'addon',
			"sources": [
				'hello.cxx'
			],
			'conditions': [
				['OS=="win"', {
						'defines': [
							'WIN'
						]
					}
				],
				['OS=="linux"', {
						'cflags': [
							'--std=c++11',
							'-pthread'
						],
						'defines': [
							'UNIX'
						]
					}

				]	
			]
		}
	]
}

```

最后就是编译： 
```
node-gyp configure
node-gyp build
```