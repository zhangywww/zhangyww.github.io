---
layout: post
title: switch_event
tag: freeswitch
---

# switch_event函数

## switch_event_create

switch_event_create用于创建一个switch_event_t事件，它实际上是switch_event_create_subclass的宏定义，
并以**SWITCH_EVENT_SUBCLASS_ANY**作为subclass名字默认参数。而switch_event_create_subclass实际上是
switch_event_create_subclass_detailed的宏定义。其关系如下

```c
#define SWITCH_EVENT_SUBCLASS_ANY   NULL

// switch_event_create
#define switch_event_create(event, id) \		   
	switch_event_create_subclass(event, id, SWITCH_EVENT_SUBCLASS_ANY)

// switch_event_create_subclass
#define switch_event_create_subclass(_e, _eid, _sn)		   
	switch_event_create_subclass_detailed(__FILE__, (const char * )__SWITCH_FUNC__, __LINE__, _e, _eid, _sn)
	
// switch_event_create_subclass_detailed
switch_status_t switch_event_create_subclass_detailed(
	const char *file,
	const char *func,
	int line,
	switch_event_t *event,
	switch_event_types_t event_id,
	const char *subclass_name 
)	
```

其中event是将要创建的switch_event结构，event_id是要创建的event类型id。

在switch_event_create_subclass_detailed函数中，当event_id为**SWITCH_EVENT_CLONE**和**SWITCH_EVENT_CUSTOM**时，
可以传入非空的subclass_name，其他的event_id只能传入NULL的subclass_name，即SWITCH_EVENT_SUBCLASS_ANY，
否则会返回SWITCH_STATUS_GENERR，用于说明创建switch_event失败。

如果程序中定义了**SWITCH_EVENT_RECYCLE**宏，switch_event会通过EVENT_RECYCLE_QUEUE队列进行复用。
没定义该宏时，会在堆中申请内存。

如果要创建的event_id为**SWITCH_EVENT_REQUEST_PARAMS**、**SWITCH_EVENT_CHANNEL_DATA**或**SWITCH_EVENT_MESSAGE**，
那么将会设置event->flags的**EF_UNIQ_HEADERS**标识，说明将创建的switch_event结构不能存在重复key的header。
(channel->variables是event_id为**SWITCH_EVENT_CHANNEL_DATA**的switch_event结构)

如果event_id不是SWITCH_EVENT_CLONE，那么将创建的switch_event的event_id设置为传入的event_id，并设置一些基本的
header。如下代码所示

```c
// switch_event.c #743
if (event_id != SWITCH_EVENT_CLONE) {
	(*event)->event_id = event_id;
	switch_event_prep_for_delivery_detailed(file, func, line, *event);
}

// switch_event.c #1929
// @f switch_event_prep_for_delivery_detailed
void switch_event_prep_for_delivery_detailed(	
	const char *file,
	const char *func,
	int line,
	switch_event_t *event)
{
	switch_time_exp_t tm;
	char date[80] = "";
	switch_size_t retsize;
	switch_time_t ts = switch_micro_time_now();
	uint64_t seq;

	switch_mutex_lock(EVENT_QUEUE_MUTEX);
	seq = ++EVENT_SEQUENCE_NR;
	switch_mutex_unlock(EVENT_QUEUE_MUTEX);

	switch_event_add_header_string(event, SWITCH_STACK_BOTTOM, "Event-Name", switch_event_name(event->event_id));
	switch_event_add_header_string(event, SWITCH_STACK_BOTTOM, "Core-UUID", switch_core_get_uuid());
	switch_event_add_header_string(event, SWITCH_STACK_BOTTOM, "FreeSWITCH-Hostname", switch_core_get_hostname());
	switch_event_add_header_string(event, SWITCH_STACK_BOTTOM, "FreeSWITCH-Switchname", switch_core_get_switchname());
	switch_event_add_header_string(event, SWITCH_STACK_BOTTOM, "FreeSWITCH-IPv4", guess_ip_v4);
	switch_event_add_header_string(event, SWITCH_STACK_BOTTOM, "FreeSWITCH-IPv6", guess_ip_v6);

	switch_time_exp_lt(&tm, ts);
	switch_strftime_nocheck(date, &retsize, sizeof(date), "%Y-%m-%d %T", &tm);
	switch_event_add_header_string(event, SWITCH_STACK_BOTTOM, "Event-Date-Local", date);
	switch_rfc822_date(date, ts);
	switch_event_add_header_string(event, SWITCH_STACK_BOTTOM, "Event-Date-GMT", date);
	switch_event_add_header(event, SWITCH_STACK_BOTTOM, "Event-Date-Timestamp", "%" SWITCH_UINT64_T_FMT, (uint64_t) ts);
	switch_event_add_header_string(event, SWITCH_STACK_BOTTOM, "Event-Calling-File", switch_cut_path(file));
	switch_event_add_header_string(event, SWITCH_STACK_BOTTOM, "Event-Calling-Function", func);
	switch_event_add_header(event, SWITCH_STACK_BOTTOM, "Event-Calling-Line-Number", "%d", line);
	switch_event_add_header(event, SWITCH_STACK_BOTTOM, "Event-Sequence", "%" SWITCH_UINT64_T_FMT, seq);
}
```

最后，如果传入的subclass_name非空，设置event的subclass_name，并添加Event-Subclass的header。
header可以理解为存储key，value的单向链表结构。

## switch_event_fire

switch_event_fire用于将switch_event结构，即event事件放入到事件队列中进行分发。switch_event_fire是
switch_event_fire_detailed的宏定义

```c
#define switch_event_fire(event) \
	switch_event_fire_detailed(__FILE__, (const char * )__SWITCH_FUNC__, __LINE__, event, NULL)

// switch_event.c #1963
switch_status_t switch_event_fire_detailed(	
	const char *file,
	const char *func,
	int line,
	switch_event_t **event,
	void *user_data
)	
```

在switch_event_fire_detailed函数中，判断runtime.events_use_dispatch是否设置，如果设置了，
会检查事件分发队列是否有创建，事件分发线程是否开启，如果没有，就创建分发队列，
并开始事件分发线程。然后将传入的switch_event结构扔到队列中。

在扔进去之前，其实还会去检查队列是否满了，如果满了，说明队列消费得太慢了，
那么就可能再创建多一个分发线程，提高消费得速度。

如果runtime.events_use_dispatch没有设置，那么会将事件放进线程池得执行队列中，由线程池进行分发。

只要事件放在了队列中，那么传进来得switch_event_t结构就会设置成空。如下
```c
*event = NULL
```
这样，只要我们调用switch_event_fire成功，我们就不能改动事件中的数据。

在系统中，当发送和channel有关的事件前，都会调用**switch_channel_event_set_data(channel, event)**
在event中设置一些和channel有关的header数据，然后再发送出去。
