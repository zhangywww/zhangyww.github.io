---
layout: post
title: switch_utils
tag: freeswitch
---

# 数据结构

switch_utils中声明的union和struct

## union ip_t (ip类型)

```c
typedef union{
	uint_32_t v4;		//ipv4地址
	struct in6_addr v6;	//ipv6地址
} ip_t

//其中ip_t中的struct in6_addr

```

## struct switch_http_request_s

```c
typedef struct switch_http_request_s {
	const char *method;		// GET POST PUT DELETE OPTIONS PATCH HEAD
	const char *uri;
	const char *qs;			// query string
	const char *host;
	switch_port_t port;
	const char *from;
	const char *user_agent;
	const char *referer;
	const char *user;
	switch_bool_t keepalive;
	const char *content_type;
	switch_size_t content_length;
	switch_size_t bytes_header;
	switch_size_t bytes_read;
	switch_size_t bytes_buffered;
	switch_size_t *headers;
	void *user_data;
	
	//用于解析器
	char *_buffer;
	switch_bool_t _destroy_headers;
}
```

## struct switch_cputime (程序运行时间)

```c
typedef struct {
	int64_t userms;
	int64_t kernelms;
} switch_cputime;
```