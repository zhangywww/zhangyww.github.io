---
layout: post
title: switch_channel
tag: freeswitch
---

# switch_channel中的函数

## switch_channel_set_variable

switch_channel_set_variable用于设置channel的通道变量。
switch_channel_set_variable是一个宏，定义如下：
```c
#define switch_channel_set_variable(_channel, _var, _val) \
	switch_channel_set_variable_var_check(_channel, _var, _val, SWITCH_TRUE)
	
// switch_channel_set_variable_var_check的完整定义如下
switch_status_t switch_channel_set_variable_var_check(
	switch_channel_t * 	channel,
	const char * 	varname,
	const char * 	value,
	switch_bool_t 	var_check 
)	
```

switch_channel_set_variable_var_check中，**channel**为对应的通道，
**vaname**和**value**为需要设置的key和value，**var_check**是一个
布尔值，用于设置是否对value进行变量检测，如果需要检测, 而且value
中含有变量(含有连续的**${** 就认为含有变量)，那么就认为该value是
无效的，而且**不会**设置该通道变量，但是会返回**SWITCH_STATUS_SUCCESS**。
并打印一个CRTI级别的日志(Invalid data (key contains a variable)\n")


对switch_channel_set_variable的调用总结：
- 所有的通道变量存储在**channel->variables**结构中(一个switch_event结构体指针)。
  每个通道变量都是以**switch_event_header**的结构进行存储，并用单向链表串起来。
- 设置通道变量过程中会使用**channel->profile_mutex**进行加锁。
- 只有当传入的key为**NULL**或**空字符串**时，返回**SWITCH_STATUS_FALSE**，
  其他情况返回**SWITCH_STATUS_SUCCESS**。
- switch_channel_set_variable会对value进行变量检测，value中不
  能含有连续的 **${**，否则设置通道变量失败，但依然会返回**SWITCH_STATUS_SUCCESS**。
- 传入的value为**NULL**或**空字符串**时，会删除对应key的通道变量。
- 设置的value值，是字符串的内存拷贝。
- 当传入的key为**_body**时，会同时设置event中的body。
- 支持存储数组。
  当key中间含有**[**时，会以**[**左侧为key值，右侧为索引，而且数组的大小时自动增长的，
  未设置的值为空字符串。数组索引的最大值为4000，超过这个值设置失败。
  
- 支持value为数组。value若以**ARRAY::**开头，则value为数组，相当于在key这个数组中追加值，
  **ARRAY::**后每个追加的值以**|:**分隔。在存储时，处理真正存储为数组外，其value中就以
  这种字符串形式进行存储。
- 以**SWITCH_STACK_BOTTOM**的形式在尾部进行增加到**switch_event_header**的链表尾部。
- **switch_event_header**有一个key的整数hash值，用于提高查询速度。
- 会根据**channel->variables**这个switch_event结构是否含有EF_UNIQ_HEADERS标识，来对重复key的处理。

## switch_channel_get_variable

switch_channel_get_variable用于获取channel中通道变量的值。switch_channel_get_variable
实际是一个宏，其定义为
```c
#define switch_channel_get_variable(_c,_v )	\	   
	switch_channel_get_variable_dup(_c, _v, SWITCH_TRUE, -1)

//switch_channel_get_variable_dup完整定义为
const char* switch_channel_get_variable_dup	(	
	switch_channel_t * 	channel,
	const char * 	varname,
	switch_bool_t 	dup,
	int 	idx 
)	
```

switch_channel_get_variable_dup中传如的dup值为SWITCH_TRUE，说明获取的是value值的拷贝
(通过session的pool分配内存)，而传的idx值为-1，表示不获取数组中的值，而是获取数组的字符串表示。

对switch_channel_get_variable总结如下

- 首先在**channel->scope_variables**(switch_event结构的链表)中尝试找通道变量，
  找不到再到**channel->variables**(单个switch_event)中找，再找不到的话，获取
  该channel的**channel->caller_profile->hunt_caller_profile**(switch_caller_profile结构)或
  **channel->caller_profile**(switch_caller_profile结构)，如果key以**aleg_**开头，
  以**aleg_**后的作为key在**profile->originator_caller_profile**中查找对应的变量，
  如果key以**bleg_**开头，以**bleg_**后的作为key在**profile->originatee_caller_profile**
  查找对应的变量。如果还是找不到，那么就通过switch_core_get_variable_pdup获取系统变量值。
- 不能数组索引的方式获取值，只能通过数组key获取数组的字符串表示。
- key为**_body**可以获取第一个非NULL的switch_event的body值。