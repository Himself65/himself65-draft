---
title: N-API初探
tags: nodejs javascript c++
---

# 前言

Node.js（下文称作Node）在 v8.x 版本中加入了 Native-API，意在独立于底层代码（如v8），这样可以完美支持C++插件在Node不同版本中使用。因为官方给出的实例不够明确，本文简单介绍并入门N-API。

## 初始化项目

```bash
yarn global add node-gyp
mkdir article-napi
cd article-napi
yarn init
yarn add node-gyp
yarn add --dev jest node-addon-api
```

因为为了方便测试，我们这里采用jest进行TDD测试，然后在 `package.json` 中加入一个指令

```json
{
  "scripts": {
    "test": "node-gyp rebuild && jest -i"
  }
}
```

然后我们新建文件 `binding.gyp`，此文件相当于 `makefile`, `dockfile` 一样的存在。

```gyp
{
  "targets": [
    {
      "target_name": "test",
      "sources": [
        "./lib/index.cc"
      ]
    }
  ]
}
```

在 `./lib/index.cc` 中创建我们的第一个c++文件

```cpp
#include <node_api.h>

napi_value Init(napi_env env, napi_value exports) {
	// exports 的绑定在这里
	return exports;
}

NAPI_MODULE(NODE_GYP_MODULE_NAME, Init)
```

比较屑的一点，napi中基本的类型其实都是 `void *`，但是给出的头文件中隐藏了这一点，所以我们不知道这些内部到底有什么东西。一方面限制了我们的操作，必须用n-api操作n-api-value；一方面也将此C++插件与底层C++隔离，实现了真正意义上的 ~~`naive`~~ `native`。

源代码如下：

```cpp
// JSVM API types are all opaque pointers for ABI stability
// typedef undefined structs instead of void* for compile time type safety
typedef struct napi_env__ *napi_env;
typedef struct napi_value__ *napi_value;
typedef struct napi_ref__ *napi_ref;
typedef struct napi_handle_scope__ *napi_handle_scope;
typedef struct napi_escapable_handle_scope__ *napi_escapable_handle_scope;
typedef struct napi_callback_info__ *napi_callback_info;
typedef struct napi_async_context__ *napi_async_context;
typedef struct napi_async_work__ *napi_async_work;
typedef struct napi_deferred__ *napi_deferred;
```

`exports` 本质上也是一个 `object`， 所以我们用 `napi_define_properties`, `napi_set_named_property` 等函数进行函数绑定。使用前者时候我们需要将函数包装到 `napi_property_descriptor` 中

```cpp
typedef struct {
  // One of utf8name or name should be NULL.
  const char* utf8name;
  napi_value name;

  napi_callback method;
  napi_callback getter;
  napi_callback setter;
  napi_value value;

  napi_property_attributes attributes;
  void* data;
} napi_property_descriptor;
```

使用宏定义更加方便

```cpp
#define DECLARE_NAPI_PROPERTY(name, func)                                \
  { (name), NULL, (func), NULL, NULL, NULL, napi_default, NULL }

napi_value Init(napi_env env, napi_value exports) {
	napi_property_descriptor desc[] = {
		DECLARE_NAPI_PROPERTY("func1", Func1),
		DECLARE_NAPI_PROPERTY("func2", Func2),
	};
	NAPI_CALL(env, napi_define_properties(env, exports, 2, desc));
	return exports;
}
```

或者使用后者时，我们需要将函数包装成 `napi_value`

```cpp
napi_value init(napi_env env, napi_value exports) {
  napi_value fn;

  napi_create_function(env, nullptr, 0, Method, nullptr, &fn);

  napi_set_named_property(env, exports, "hello", fn);
  return exports;
}
```

为了方便拓展，我们新建一个 `index.js` 转发一下

```js
module.exports = require('./build/Release/test')
```

```cpp
#include <node_api.h>

#define NAPI_ASSERT_BASE(env, assertion, message, ret_val)               \
  do {                                                                   \
    if (!(assertion)) {                                                  \
      napi_throw_error(                                                  \
          (env),                                                         \
        NULL,                                                            \
          "assertion (" #assertion ") failed: " message);                \
      return ret_val;                                                    \
    }                                                                    \
  } while (0)

#define NAPI_ASSERT(env, assertion, message)                             \
  NAPI_ASSERT_BASE(env, assertion, message, NULL)

#define DECLARE_NAPI_PROPERTY(name, func)                                \
  { (name), NULL, (func), NULL, NULL, NULL, napi_default, NULL }

struct MyObject {
	int32_t val;
};

static napi_value Create(napi_env env, napi_callback_info info) {
	size_t argc = 1;
	napi_value args[1];

	napi_get_cb_info(env, info, &argc, args, NULL, NULL);
	NAPI_ASSERT(env, argc >= 1, "Wrong number of arguments");
	napi_valuetype valuetype;
	napi_typeof(env, args[0], &valuetype);
	NAPI_ASSERT(
		env, valuetype == napi_number, "Wrong argument type. Number expected");

	MyObject* external = new MyObject();
	napi_get_value_int32(env, args[0], &(external->val));
	napi_value val;
	napi_create_external(env, external, nullptr, nullptr, &val);
	return val;
}

static napi_value ToInt32(napi_env env, napi_callback_info info) {
	size_t argc = 1;
	napi_value args[1];

	napi_get_cb_info(env, info, &argc, args, NULL, NULL);
	NAPI_ASSERT(env, argc >= 1, "Wrong number of arguments");
	napi_valuetype valuetype;
	napi_typeof(env, args[0], &valuetype);
	NAPI_ASSERT(env,
		valuetype == napi_external,
		"Wrong argument type. External expected");
	MyObject* external;
	napi_value val;
	napi_get_value_external(env, args[0], reinterpret_cast<void**>(&external));
	napi_create_int32(env, external->val, &val);
	return val;
}

napi_value Init(napi_env env, napi_value exports) {
	napi_property_descriptor desc[] = {
		DECLARE_NAPI_PROPERTY("createExternal", Create),
		DECLARE_NAPI_PROPERTY("getVal", ToInt32),
	};
	napi_define_properties(env, exports, 2, desc);
	return exports;
}

NAPI_MODULE(NODE_GYP_MODULE_NAME, Init)
```

测试就可以开始写了，如下

```js
// test/index.test.js
const test = require('../')
const util = require('util')

describe('Unit text', () => {
	it('should success', () => {
		const external = test.createExternal(1000)
		expect(util.types.isExternal(external)).toBe(true)
		expect(test.getVal(external)).toBe(1000)
	})
})
```

```ts
// index.d.ts
export function createExternal (val: number): Object

export function getVal (val: Object): number
```

---

## 推荐阅读

[1] [《Node.js：来一打 C++ 扩展》已出版，求支持](https://xcoder.in/2018/06/14/node-cpp-advanced-sale/)

[2] [Nodejs v12.4.0官方文档](https://nodejs.org/dist/latest-v12.x/docs/api/n-api.html)

[3] [Nodejs GitHub源码](https://github.com/nodejs/node/tree/master/test/js-native-api)
