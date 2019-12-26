---
layout: post
title: docker默认bridge0
tag: docker
---

# docker0

**docker0**是docker默认提供的bridge。在docker服务启动的时候，**docker0**也会被启动起来。

```shell
##> brctl show

bridge name	bridge id		STP enabled	interfaces
docker0		8000.024274c081f4	no		
```

docker为**bridge0**这个linux bridge创建了一个名为**bridge**的network。通过docker network ls可以显示docker创建的network。

```shell
##> docker network ls

NETWORK ID          NAME                DRIVER              SCOPE
4eb24ce2f823        bridge              bridge              local
e3f8abebb260        host                host                local
6d5552fd0d8c        none                null                local
```

通过docker network inspect可以进一步查看docker所创建的network

```json
##> docker network inspect bridge

[
    {
        "Name": "bridge",
        "Id": "4eb24ce2f823390982e59f5ab5fdbde22c5b03d7e58899522aeb65cd6462cf18",
        "Created": "2019-12-22T21:22:05.936932523-05:00",
        "Scope": "local",
        "Driver": "bridge",
        "EnableIPv6": false,
        "IPAM": {
            "Driver": "default",
            "Options": null,
            "Config": [
                {
                    "Subnet": "172.17.0.0/16"
                }
            ]
        },
        "Internal": false,
        "Attachable": false,
        "Containers": {},
        "Options": {
            "com.docker.network.bridge.default_bridge": "true",
            "com.docker.network.bridge.enable_icc": "true",
            "com.docker.network.bridge.enable_ip_masquerade": "true",
            "com.docker.network.bridge.host_binding_ipv4": "0.0.0.0",
            "com.docker.network.bridge.name": "docker0",
            "com.docker.network.driver.mtu": "1500"
        },
        "Labels": {}
    }
]
```



# 参考

> [Networking your docker containers using docker0 bridge](https://developer.ibm.com/recipes/tutorials/networking-your-docker-containers-using-docker0-bridge/)
> [Understanding Docker Networking Drivers and their use cases](https://www.docker.com/blog/understanding-docker-networking-drivers-use-cases/)
