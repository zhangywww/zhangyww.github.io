---
layout: post
title: shell字符串处理
tag: shell
---

# awk

```shell
echo '{"key1":"value1","key2":"value2","key3":"value3","key4":"value4"}' | \
awk '{match($0, /key1":"(\w+[^"]).*key3":"(\w+[^"])/, a); print a[1], a[2]}'
```