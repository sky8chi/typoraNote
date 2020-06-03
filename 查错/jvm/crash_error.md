# Crashes in ZIP_GetEntry

```shell
#
#  SIGBUS (0x7) at pc=0x00000034fec897cb, pid=225034, tid=140207454631680
#
# JRE version: Java(TM) SE Runtime Environment (7.0_80-b15) (build 1.7.0_80-b15)
# Java VM: Java HotSpot(TM) 64-Bit Server VM (24.80-b11 mixed mode linux-amd64 compressed oops)
# Problematic frame:
# C  [libc.so.6+0x897cb]  memcpy+0x15b

```

Jvm在运行时加载jar包有一个优化，他会在内存中存储jar文件内的目录结构，所以如果你在运行过程中动态替换jar包导致内存中的副本与实际不一致会crash掉 (自行搜索mmap原理)

可以增加去除优化（会带来性能损耗）

```shell
-Dsun.zip.disableMemoryMapping=true
```

> https://blog.csdn.net/qq_33611327/article/details/81738195
>
>  https://blogs.oracle.com/poonam/crashes-in-zipgetentry