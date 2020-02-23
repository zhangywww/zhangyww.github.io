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

**CS_INIT**状态表明这个channel已经初始化，