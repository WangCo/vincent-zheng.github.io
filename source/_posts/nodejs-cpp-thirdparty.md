title: node c++ 调用第三方库之TinyXPath
---
> 本程序使用c++的第三方库TinyXPath进行实验，以探究c++调用第三方库的方法，TinyXPath官网可以点击[这里](http://tinyxpath.sourceforge.net/)查看。

## 简述
本程序主要参照[nodejs官方文档](https://nodejs.org/documentation/)和部分v8引擎源码（这里我也正在看，所以可能会有比较多的误差）。程序完整源码可以点击[这里](https://github.com/vincent-zheng/xpath4js)获取。

## 编译环境
本程序在VS2013和gcc4.9.2下编译通过，nodejs版本为0.12.4。

## 完整源码解析

### main.cxx
程序主入口文件， 用以初始化类，本程序使用nodejs里面的WrapObject的方式来实现用c++代码构建js对象。下面只是调用XPath这个类里面的Init函数初始化对象，所以没什么好说的了 ~~
```
// main.cxx
#include "xpath4js.h"
using namespace v8;
using namespace std;

void init(Handle<Object> exports) {
	XPath::Init(exports);
}

NODE_MODULE(xpath4js, init)
```
 
### xpath4js.h
XPath对象头文件，定义的Init函数用以初始化对象。调用TinyXPath这个第三方库只需要包含`xpath_static.h`这个头文件即可，TinyXPath官方文档上面是说明TinyXPath这个库是先通过编译成静态库然后调用的，而我这里为了方便同时支持windows和linux平台，所以将这个编译的方式改成了直接编译源文件，在后面的binding.gyp文件中会有说明。需要另外说明的是对象中需要包含在exports出去的js对象中的话需要使用静态函数，当然，需要调用到的变量不需要这样做，不过记得要在构造函数中构造好。

```
#ifndef XPATH4JS
#define XPATH4JS

#include "../lib/tinyxpath_1_3_1/xpath_static.h"
#include <node.h>
#include <node_object_wrap.h>
#include <iostream>

class XPath : public node::ObjectWrap {
public:
	static void Init(v8::Handle<v8::Object> exports);
private:
	explicit XPath(const char * xml);
	~XPath();

	static void New(const v8::FunctionCallbackInfo<v8::Value> & args);
	static void parse(const v8::FunctionCallbackInfo<v8::Value> & args);
	static void get(const v8::FunctionCallbackInfo<v8::Value> & args);
	static v8::Persistent<v8::Function> constructor;

	char * _xml;
	TiXmlDocument * doc;
};

#endif 
```

### xpath4js.cxx

为了方便说明，这个文件的说明直接写在相应代码行的注释中。

```
#include "xpath4js.h"
using namespace v8;
using namespace std;

Persistent<Function> XPath::constructor;

// 用以获取char *字符串的长度
int charLength(const char * str) {
	int i = 0;
	while (str[i] != '\0') {
		i++;
	}
	return i;
}

// 构造函数， 配置相应的非静态变量
XPath::XPath(const char * xml): _xml(xml) {
	doc = new TiXmlDocument;
	if (charLength(xml)) {
		doc->Parse(xml);
	}
}

XPath::~XPath() {

}

// xports的初始化方法， 将c++对象转化为js对象
void XPath::Init(Handle<Object> exports) {
    // 当然是获取当前的Isolate句柄先
	Isolate * isolate = Isolate::GetCurrent();

    /* JS中的对象就是使用函数的方式实现的，而c++这里也是一样， 
       使用FunctionTemplate这个对象构造出相应的函数，然后对
       他的原型链进行相应的函数赋值，这里的函数是调用XPath这
       个类里面的New函数作为构造函数的。
    */
	Local<FunctionTemplate> tpl = FunctionTemplate::New(isolate, New);
	tpl->SetClassName(String::NewFromUtf8(isolate, "XPath"));
	tpl->InstanceTemplate()->SetInternalFieldCount(1);

    // 将函数插入原型链中
	NODE_SET_PROTOTYPE_METHOD(tpl, "parse", parse);
	NODE_SET_PROTOTYPE_METHOD(tpl, "get", get);
	constructor.Reset(isolate, tpl->GetFunction());
    // exports出对象
	exports->Set(String::NewFromUtf8(isolate, "XPath"), tpl->GetFunction());
}

// 对象的构造方法
void XPath::New(const FunctionCallbackInfo<Value> & args) {
	Isolate * isolate = Isolate::GetCurrent();
	HandleScope scope(isolate);
	XPath * instance;
    // 这里不是很确定，是模仿nodejs官档上面的做法的，我猜测是判断在js调用的时候是否为使用new构造。
	if (args.IsConstructCall()) {
		// 我原以为会有IsString方法，但是查看了一下v8里面的源码，发现是没有的~~ ， 我猜测是因为js中所有对象都是可以变成String类型的，所以不需要强加判断
		if (args.Length() > 0 /* && args[0].IsString()*/) {
            // 获取参数并转化为const char * 类型
			String::Utf8Value str(args[0]);
			const char * xml = * str;
            // 实例化XPath对象
			instance = new XPath(xml);
		} else {
			instance = new XPath("");
		}
        // 将对象传递出去
		instance->Wrap(args.This());
		args.GetReturnValue().Set(args.This());
	} else {
        // 这里面是如果不是使用new 构造就将其内部转化成使用new构造时的方式。
		Local<Function> func = Local<Function>::New(isolate, constructor);
		if (args.Length() > 0 /* && args[0].IsString()*/) {
			String::Utf8Value str(args[0]);
			const char * xml = * str;
			Local<Value> argv[1] = {String::NewFromUtf8(isolate, xml)};
			args.GetReturnValue().Set(func->NewInstance(1, argv));
		} else {
			Local<Value> argv[1] = {String::NewFromUtf8(isolate, "")};
			args.GetReturnValue().Set(func->NewInstance(1, argv));
		}
	}
}

// 定义了一个parse方法，用于传入需要用XPath解析的文档
void XPath::parse(const v8::FunctionCallbackInfo<v8::Value> & args) {
	Isolate * isolate = Isolate::GetCurrent();
	HandleScope scope(isolate);

	XPath * instance = ObjectWrap::Unwrap<XPath> (args.Holder());
	if (args.Length() > 0  /* && args[0].IsString()*/) {
		String::Utf8Value str(args[0]);
		const char * xml = *str;
		instance->_xml = xml;
		instance->doc->Parse(instance->_xml);
		args.GetReturnValue().Set(Boolean::New(isolate, true));
	} else {
		args.GetReturnValue().Set(Boolean::New(isolate, false));
	}
}

// 使用get方法获取XPath路径下的内容。
void XPath::get(const v8::FunctionCallbackInfo<v8::Value> & args) {
	Isolate * isolate = Isolate::GetCurrent();
	HandleScope scope(isolate);

	XPath * instance = ObjectWrap::Unwrap<XPath>(args.Holder());
	if (args.Length() > 0) {
		String::Utf8Value str(args[0]);
		const char * path = *str;
		TIXML_STRING S_res = TinyXPath::S_xpath_string(instance->doc->RootElement(), path);
		Local<String> result = String::NewFromUtf8(isolate, S_res.c_str());
		args.GetReturnValue().Set(result);
	} else {
		args.GetReturnValue().Set(String::NewFromUtf8(isolate, ""));
	}
}
```

### test.js

测试代码

```
var xpath4js = require('./../build/Release/xpath4js');

var xml = "<?xml version=\"1.0\" encoding=\"UTF-8\"?>" +
           	"<root>" +
                "<child foo='bar'>" +
                    "<grandchild baz=\"fizbuzz\">grandchild content</grandchild>" +
                "</child>" +
                "<sibling>with content!</sibling>" + 
           	"</root>";

var xpath = new xpath4js.XPath();
xpath.parse(xml);
console.log(xpath.get('//grandchild/text()'));

```