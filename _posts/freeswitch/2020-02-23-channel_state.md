---
layout: post
title: channel_state
tag: freeswitch
---

# channel state 

当建立起一个channel的时候，freeswitch会以状态机的方式去维护这个channel的状态，channel可能的状态为

```c
// switch_types.h #1307
typedef enum {
	CS_NEW,
	CS_INIT,
	CS_ROUTING,
	CS_SOFT_EXECUTE,
	CS_EXECUTE,
	CS_EXCHANGE_MEDIA,
	CS_PARK,
	CS_CONSUME_MEDIA,
	CS_HIBERNATE,
	CS_RESET,
	CS_HANGUP,
	CS_REPORTING,
	CS_DESTROY,
	CS_NONE
} switch_channel_state_t;
```

这些状态也会对应其相应的常量字符串表示

```c
// switch_channel.c #2154
static const char *state_names[] = {
	"CS_NEW",
	"CS_INIT",
	"CS_ROUTING",
	"CS_SOFT_EXECUTE",
	"CS_EXECUTE",
	"CS_EXCHANGE_MEDIA",
	"CS_PARK",
	"CS_CONSUME_MEDIA",
	"CS_HIBERNATE",
	"CS_RESET",
	"CS_HANGUP",
	"CS_REPORTING",
	"CS_DESTROY",
	"CS_NONE",
	NULL
};
```

# channel state的变化时刻

## CS_NEW

**CS_NEW**状态表明这个channel刚创建出来，其在switch_core_session_request函数创建session
的时候设置。

```c
//switch_core_session.c 
// @f switch_core_session_request_uuid #2236
switch_channel_init(session->channel, session, CS_NEW, 0);
```

## CS_INIT

**CS_INIT**状态表明这个channel已经初始化。如果使用mod_sofia作为endpoint模块，在进行**外呼**时，
我们可以跟一下sofia_endpoint_interface需要实现io_routinue中外呼回调函数，在mod_sofia模块中，其实现为
**sofia_outgoing_channel**函数。

在sofia_outgoing_channel中，可以看到下面的代码
```c
// mod_sofia.c #4457
if (!(nsession = switch_core_session_request_uuid(sofia_endpoint_interface, SWITCH_CALL_DIRECTION_OUTBOUND,
												  flags, pool, switch_event_get_header(var_event, "origination_uuid")))) {
	switch_log_printf(SWITCH_CHANNEL_LOG, SWITCH_LOG_CRIT, "Error Creating Session\n");
	goto error;
}
```
通过switch_core_session_request_uuid创建了一个新的session，建立了一个新的channel，在该函数中，
channel的state被初始化为了**CS_NEW**。

接着往下看的话，可以看到
```c
// mod_sofia.c #4815
sofia_set_flag_locked(tech_pvt, TFLAG_OUTBOUND);
sofia_clear_flag_locked(tech_pvt, TFLAG_LATE_NEGOTIATION);
if (switch_channel_get_state(nchannel) == CS_NEW) {
	switch_channel_set_state(nchannel, CS_INIT);
}
tech_pvt->caller_profile = caller_profile;
*new_session = nsession;
```

在对新建立的channel建立一系列的初始化之后，将channel的state设置成了**CS_INIT**，
并将nsession赋值给了传进来的**new_session**，用于返回。