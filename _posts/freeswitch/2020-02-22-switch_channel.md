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
  因为在创建session的时候，session的channel->variables的eventid设置成了SWITCH_EVENT_CHANNEL_DATA,
  而SWITCH_EVENT_REQUEST_PARAMS 和 variables的eventid设置成了 SWITCH_EVENT_CHANNEL_DATA 这两种switch_event类型
  都会设置**EF_UNIQ_HEADERS**的flags. 因此设置通道变量是能重复key的，重复会删除先前的再插入新的。
  (在通过switch_event_create函数创建event时，若event_id指定为而**SWITCH_EVENT_REQUEST_PARAMS**，
  **SWITCH_EVENT_CHANNEL_DATA** 或 **SWITCH_EVENT_MESSAGE**三个中的其中一个，都会设置event->flags增加
  **EF_UNIQ_HEADERS**。

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
- 获取通道变量过程中会使用**channel->profile_mutex**进行加锁。
- 不能数组索引的方式获取值，只能通过数组key获取数组的字符串表示。
- key为**_body**可以获取第一个非NULL的switch_event的body值。

## switch_channel_event_set_data

在系统中，当发送和channel有关的事件前，如**SWITCH_EVENT_CHANNEL_ANSWER**，
都会调用**switch_channel_event_set_data(channel, event)**
在event中设置一些和channel有关的header数据，然后再发送出去。
switch_channel_event_set_data的函数代码如下

```c
	switch_mutex_lock(channel->profile_mutex);
	switch_channel_event_set_basic_data(channel, event);
	switch_channel_event_set_extended_data(channel, event);
	switch_mutex_unlock(channel->profile_mutex);
```

可见，switch_channel_event_set_data给event设置了两种类型的数据，一种是**basic_data**，
另一种是**extended_data**。

其中**basic_data**主要设置每个channel基本都可能有的header数据，其代码如下

```c
// switch_channel.c #2507
// SWITCH_DECLARE(void) switch_channel_event_set_basic_data... 为了代码高亮...
void switch_channel_event_set_basic_data(switch_channel_t *channel, switch_event_t *event)
{
	switch_caller_profile_t *caller_profile, *originator_caller_profile = NULL, *originatee_caller_profile = NULL;
	switch_codec_implementation_t impl = { 0 };
	char state_num[25];
	const char *v;

	switch_mutex_lock(channel->profile_mutex);

	if ((caller_profile = channel->caller_profile)) {
		originator_caller_profile = caller_profile->originator_caller_profile;
		originatee_caller_profile = caller_profile->originatee_caller_profile;
	}

	switch_event_add_header_string(event, SWITCH_STACK_BOTTOM, "Channel-State", switch_channel_state_name(channel->running_state));
	switch_event_add_header_string(event, SWITCH_STACK_BOTTOM, "Channel-Call-State", switch_channel_callstate2str(channel->callstate));
	switch_snprintf(state_num, sizeof(state_num), "%d", channel->state);
	switch_event_add_header_string(event, SWITCH_STACK_BOTTOM, "Channel-State-Number", state_num);
	switch_event_add_header_string(event, SWITCH_STACK_BOTTOM, "Channel-Name", switch_channel_get_name(channel));
	switch_event_add_header_string(event, SWITCH_STACK_BOTTOM, "Unique-ID", switch_core_session_get_uuid(channel->session));

	switch_event_add_header_string(event, SWITCH_STACK_BOTTOM, "Call-Direction",
								   channel->direction == SWITCH_CALL_DIRECTION_OUTBOUND ? "outbound" : "inbound");
	switch_event_add_header_string(event, SWITCH_STACK_BOTTOM, "Presence-Call-Direction",
								   channel->direction == SWITCH_CALL_DIRECTION_OUTBOUND ? "outbound" : "inbound");

	switch_event_add_header_string(event, SWITCH_STACK_BOTTOM, "Channel-HIT-Dialplan", 
								   switch_channel_direction(channel) == SWITCH_CALL_DIRECTION_INBOUND ||
								   switch_channel_test_flag(channel, CF_DIALPLAN) ? "true" : "false");


	if ((v = switch_channel_get_variable_dup(channel, "presence_id", SWITCH_FALSE, -1))) {
		switch_event_add_header_string(event, SWITCH_STACK_BOTTOM, "Channel-Presence-ID", v);
	}

	if ((v = switch_channel_get_variable_dup(channel, "presence_data", SWITCH_FALSE, -1))) {
		switch_event_add_header_string(event, SWITCH_STACK_BOTTOM, "Channel-Presence-Data", v);
	}


	if ((v = switch_channel_get_variable_dup(channel, "presence_data_cols", SWITCH_FALSE, -1))) {
		switch_event_add_header_string(event, SWITCH_STACK_BOTTOM, "Presence-Data-Cols", v);
		switch_event_add_presence_data_cols(channel, event, "PD-");
	}

	if ((v = switch_channel_get_variable_dup(channel, "call_uuid", SWITCH_FALSE, -1))) {
		switch_event_add_header_string(event, SWITCH_STACK_BOTTOM, "Channel-Call-UUID", v);
	} else {
		switch_event_add_header_string(event, SWITCH_STACK_BOTTOM, "Channel-Call-UUID", switch_core_session_get_uuid(channel->session));
	}

	if (switch_channel_down_nosig(channel)) {
		switch_event_add_header_string(event, SWITCH_STACK_BOTTOM, "Answer-State", "hangup");
	} else if (switch_channel_test_flag(channel, CF_ANSWERED)) {
		switch_event_add_header_string(event, SWITCH_STACK_BOTTOM, "Answer-State", "answered");
	} else if (switch_channel_test_flag(channel, CF_EARLY_MEDIA)) {
		switch_event_add_header_string(event, SWITCH_STACK_BOTTOM, "Answer-State", "early");
	} else {
		switch_event_add_header_string(event, SWITCH_STACK_BOTTOM, "Answer-State", "ringing");
	}

	if (channel->hangup_cause) {
		switch_event_add_header_string(event, SWITCH_STACK_BOTTOM, "Hangup-Cause", switch_channel_cause2str(channel->hangup_cause));
	}


	switch_core_session_get_read_impl(channel->session, &impl);

	if (impl.iananame) {
		switch_event_add_header_string(event, SWITCH_STACK_BOTTOM, "Channel-Read-Codec-Name", impl.iananame);
		switch_event_add_header(event, SWITCH_STACK_BOTTOM, "Channel-Read-Codec-Rate", "%u", impl.actual_samples_per_second);
		switch_event_add_header(event, SWITCH_STACK_BOTTOM, "Channel-Read-Codec-Bit-Rate", "%d", impl.bits_per_second);
	}

	switch_core_session_get_write_impl(channel->session, &impl);

	if (impl.iananame) {
		switch_event_add_header_string(event, SWITCH_STACK_BOTTOM, "Channel-Write-Codec-Name", impl.iananame);
		switch_event_add_header(event, SWITCH_STACK_BOTTOM, "Channel-Write-Codec-Rate", "%u", impl.actual_samples_per_second);
		switch_event_add_header(event, SWITCH_STACK_BOTTOM, "Channel-Write-Codec-Bit-Rate", "%d", impl.bits_per_second);
	}

	/* Index Caller's Profile */
	if (caller_profile) {
		switch_caller_profile_event_set_data(caller_profile, "Caller", event);
	}

	/* Index Originator/ee's Profile */
	if (originator_caller_profile && channel->last_profile_type == LP_ORIGINATOR) {
		switch_event_add_header_string(event, SWITCH_STACK_BOTTOM, "Other-Type", "originator");
		switch_caller_profile_event_set_data(originator_caller_profile, "Other-Leg", event);
	} else if (originatee_caller_profile && channel->last_profile_type == LP_ORIGINATEE) {	/* Index Originatee's Profile */
		switch_event_add_header_string(event, SWITCH_STACK_BOTTOM, "Other-Type", "originatee");
		switch_caller_profile_event_set_data(originatee_caller_profile, "Other-Leg", event);
	}

	switch_mutex_unlock(channel->profile_mutex);
}
```

而**extended_data**主要设置**channel->scope_variables**和**channel->variables**，源代码如下

```c
// switch_channel.c #2607
// 实际为 SWITCH_DECLARE(void) switch_channel_event_set_extended_data
void switch_channel_event_set_extended_data(switch_channel_t *channel, switch_event_t *event)
{
	switch_event_header_t *hi;
	int global_verbose_events = -1;

	switch_mutex_lock(channel->profile_mutex);

	switch_core_session_ctl(SCSC_VERBOSE_EVENTS, &global_verbose_events);

	if (global_verbose_events || 
		switch_channel_test_flag(channel, CF_VERBOSE_EVENTS) ||
		switch_event_get_header(event, "presence-data-cols") ||
		event->event_id == SWITCH_EVENT_CHANNEL_CREATE ||
		event->event_id == SWITCH_EVENT_CHANNEL_ORIGINATE ||
		event->event_id == SWITCH_EVENT_CHANNEL_UUID ||
		event->event_id == SWITCH_EVENT_CHANNEL_ANSWER ||
		event->event_id == SWITCH_EVENT_CHANNEL_PARK ||
		event->event_id == SWITCH_EVENT_CHANNEL_UNPARK ||
		event->event_id == SWITCH_EVENT_CHANNEL_BRIDGE ||
		event->event_id == SWITCH_EVENT_CHANNEL_UNBRIDGE ||
		event->event_id == SWITCH_EVENT_CHANNEL_PROGRESS ||
		event->event_id == SWITCH_EVENT_CHANNEL_PROGRESS_MEDIA ||
		event->event_id == SWITCH_EVENT_CHANNEL_HANGUP ||
		event->event_id == SWITCH_EVENT_CHANNEL_HANGUP_COMPLETE ||
		event->event_id == SWITCH_EVENT_REQUEST_PARAMS ||
		event->event_id == SWITCH_EVENT_CHANNEL_DATA ||
		event->event_id == SWITCH_EVENT_CHANNEL_EXECUTE ||
		event->event_id == SWITCH_EVENT_CHANNEL_EXECUTE_COMPLETE ||
		event->event_id == SWITCH_EVENT_CHANNEL_DESTROY ||
		event->event_id == SWITCH_EVENT_SESSION_HEARTBEAT ||
		event->event_id == SWITCH_EVENT_API ||
		event->event_id == SWITCH_EVENT_RECORD_START ||
		event->event_id == SWITCH_EVENT_RECORD_STOP || 
		event->event_id == SWITCH_EVENT_PLAYBACK_START ||
		event->event_id == SWITCH_EVENT_PLAYBACK_STOP ||
		event->event_id == SWITCH_EVENT_CALL_UPDATE || 
		event->event_id == SWITCH_EVENT_MEDIA_BUG_START || 
		event->event_id == SWITCH_EVENT_MEDIA_BUG_STOP || 
		event->event_id == SWITCH_EVENT_CHANNEL_HOLD || 
		event->event_id == SWITCH_EVENT_CHANNEL_UNHOLD || 
		event->event_id == SWITCH_EVENT_CUSTOM) {

		/* Index Variables */

		if (channel->scope_variables) {
			switch_event_t *ep;

			for (ep = channel->scope_variables; ep; ep = ep->next) {
				for (hi = ep->headers; hi; hi = hi->next) {
					char buf[1024];
					char *vvar = NULL, *vval = NULL;
					
					vvar = (char *) hi->name;
					vval = (char *) hi->value;
						
					switch_assert(vvar && vval);
					switch_snprintf(buf, sizeof(buf), "scope_variable_%s", vvar);
					
					if (!switch_event_get_header(event, buf)) {
						switch_event_add_header_string(event, SWITCH_STACK_BOTTOM, buf, vval);
					}
				}
			}
		}

		if (channel->variables) {
			for (hi = channel->variables->headers; hi; hi = hi->next) {
				char buf[1024];
				char *vvar = NULL, *vval = NULL;

				vvar = (char *) hi->name;
				vval = (char *) hi->value;
				
				switch_assert(vvar && vval);
				switch_snprintf(buf, sizeof(buf), "variable_%s", vvar);
				switch_event_add_header_string(event, SWITCH_STACK_BOTTOM, buf, vval);
			}
		}
	}

	switch_mutex_unlock(channel->profile_mutex);
```

