---
title: "了解openstack"
tags: openstack
---

- nova创建的虚拟机和docker的区别

nova创建的虚拟机基于硬件的虚拟化，与宿主机隔离，有独立的操作系统。
虚拟机运行在hypervisor之上，如linux的kvm。

docker容器共享主机的内核，使用命名空间、cgroup来实现轻量级的隔离。

- keystone的租户和用户的关系

租户是一个逻辑上的资源隔离单元，有独立的计算、存储、网络资源。

用户是对资源操作的主体。

一个租户（比如web.app）可以对应多个用户（比如开发、管理、测试），
一个用户也可以对应多个租户。
