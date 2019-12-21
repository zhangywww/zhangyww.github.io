---
layout: post
tag: freeswitch
title: 写一个freeswitch模块
---

# 模块Load, runtime和shutdown函数

每一个freeswitch模块必须至少声明和定义**LOAD**和**SHUTDOWN**函数，而**RUNTIME**函数的声明和定义不是必须的。

```c
#include <switch.h>

//函数的定义
SWITCH_MODULE_LOAD_FUNCTION(on_load_func);
SWITCH_MODULE_RUNTIME_FUNCTION(on_runtime_func);
SWITCH_MODULE_SHUTDOWN_FUNCTION(on_shutdown_func);

//函数的实现
//模块加载时执行的函数
SWITCH_MODULE_LOAD_FUNCTION(on_load_func){
	// to do
};

//LOAD函数执行结束后会起一个线程执行的函数
SWITCH_MODULE_RUNTIME_FUNCTION(on_runtime_func){
	// to do
};

//模块卸载时执行的函数
SWITCH_MODULE_SHUTDOWN_FUNCTION(on_shutdown_func){
	// to do
};
```

**LOAD**函数用于模块加载时初始化模块的资源。**SHUTDOWN**函数用于模块卸载时释放资源。
**RUNTIME**函数会在**LOAD**函数执行成功后，开一个单独的线程单独执行。

## SWITCH_MODULE_LOAD_FUNCTION

**SWITCH_MODULE_LOAD_FUNCTION**用于定义模块加载时执行的函数。该宏定义为

```c
#define SWITCH_MODULE_LOAD_FUNCTION(name) \
switch_status_t name SWITCH_MODULE_LOAD_ARGS

//其中 
#define SWITCH_MODULE_LOAD_ARGS \
(switch_loadable_module_interface_t **module_interface, switch_memory_pool_t *pool)

//因此整体展开为
#define SWITCH_MODULE_LOAD_FUNCTION(name) \
switch_status_t name (switch_loadable_module_interface_t **module_interface, switch_memory_pool_t *pool)
```

## SWITCH_MODULE_SHUTDOWN_FUNCTION

**SWITCH_MODULE_SHUTDOWN_FUNCTION**用于定义模块卸载时执行的函数。该宏定义为

```c
#define SWITCH_MODULE_SHUTDOWN_FUNCTION(name) \
switch_status_t name SWITCH_MODULE_SHUTDOWN_ARGS

//其中 
#define SWITCH_MODULE_SHUTDOWN_ARGS (void)

//所以SHUTDOWN宏整体展开为
#define SWITCH_MODULE_SHUTDOWN_FUNCTION(name) \
switch_status_t name (void)
```

## SWITCH_MODULE_RUNTIME_FUNCTION

**SWITCH_MODULE_RUNTIME_FUNCTION**用于定义模块在运行之间需要执行的操作。该宏定义为

```c
#define SWITCH_MODULE_RUNTIME_FUNCTION(name) \
switch_status_t name SWITCH_MODULE_RUNTIME_ARGS

//其中 
#define SWITCH_MODULE_RUNTIME_ARGS (void)

//所以RUNTIME宏整体展开为
#define SWITCH_MODULE_RUNTIME_FUNCTION(name) \
switch_status_t name (void)
```

# 写一个API

```c
switch_api_interface_t *api_interface = NULL;

SWITCH_ADD_API(api_interface, "example", "example description", 
	example_api_func, "");
```

其中的**switch_api_interface_t**类型是一个结构类型，SWITCH_ADD_API是一个宏。他们的定义如下。

```c
//API结构体类型
struct switch_api_interface {
	/*! the name of the interface */
	const char *interface_name;
	/*! a description of the api function */
	const char *desc;
	/*! function the api call uses */
	switch_api_function_t function;
	/*! an example of the api syntax */
	const char *syntax;
	switch_thread_rwlock_t *rwlock;
	int refs;
	switch_mutex_t *reflock;
	switch_loadable_module_interface_t *parent;
	struct switch_api_interface *next;
};

//添加API的宏
#define SWITCH_ADD_API(api_int, int_name, descript, funcptr, syntax_string) \
	for (;;) { \
	api_int = (switch_api_interface_t *)switch_loadable_module_create_interface(*module_interface, SWITCH_API_INTERFACE); \
	api_int->interface_name = int_name; \
	api_int->desc = descript; \
	api_int->function = funcptr; \
	api_int->syntax = syntax_string; \
	break; \
	}
```


# 写一个APP



