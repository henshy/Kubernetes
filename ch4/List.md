# List

### 性能瓶颈问题

#### 1、k8s持续性测试，（规则）服务有较多超时

​	规则服务本身性能低下，后期会做重构。

#### 2、K8s生产环境服务异常重启

​	pod内存不够，增加内容上限。

#### 3、测试过程中Tps和延迟（ms）瓶颈

- JMeter性能瓶颈（主要为CPU）
- 容器性能拐点

#### 4、一次启动200个pod失败

​	一次启动时，会同时走网络、磁盘io，导致阻塞。使用扩容命令，一次扩容一部分，可以解决此问题。

#### 5、服务占用cpu（同一类服务，有的pod占用）

​	同一服务下，有的pod占用cpu较高。

​	现象：内存正常，Ygc较频繁，但是Ogc正常，说明不是内存不够。cpu占用较大，则使用java线程排查问题工具。

​        结论：TreeMap并发冲突导致死循环。

### 配置问题

#### 1、一个节点断网后，中间请求还会过去

​	treafik配置问题，修改pod失效反应时间配置。

#### 2、有的节点日志丢失

​	GlusterFS没有把全部pod配置成镜像模式。

#### 3、测试过程中，pod分布不均匀

​        有可能全部eureka在一个节点，节点宕机会影响业务，启动反亲和配置。

#### 4、高并发情况下，会有几条请求出现500 Internal Server Error

​	Haproxy超时导致，超时时间为90s，Haproxy超时后，会告诉treafik，treafik返回该异常。

### linux bug

#### 1、压力测试后，隔了一天，其中一个节点宕机

​	内核bug（与linux与kubernetes结合后的问题），升级linux内核解决。

```shell
经过生产测试，Centos7.3使用kernel-3.10.0-514.16.1.el7内核在结合k8s使用的时候，且文件系统使用xfs时，当有node上大量pod运行，在执行重启docker服务时候，docker会执行如下操作卸载pod挂载的文件：
Dec 26 13:40:30 m24p207 kernel: XFS (dm-130): Ending clean mount
Dec 26 13:40:30 m24p207 kernel: XFS (dm-131): Mounting V5 Filesystem
Dec 26 13:40:30 m24p207 kernel: XFS (dm-131): Ending clean mount
Dec 26 13:40:30 m24p207 kernel: XFS (dm-131): Unmounting Filesyste

这个时候有可能会触发内核的bug，导致系统重启。具体描述地址：https://bugs.centos.org/view.php?id=14073 或者：https://access.redhat.com/solutions/2779111

根据官方提供的解决方案，需要将内核升级到： 3.10.0-693.el7 or later以上版本。所以需要升级内核，重启系统：

# yum install kernel-3.10.0-693.11.1.el7.x86_64 -y
# reboot 
```

#### 2、服务注册eureka偶发性失败

​	从日志中可以看到，连接eureka时，会偶发性失败。延长注册超时时间后，此问题消失。

```shell
CentOS 6/CentOS 7中的DNS解析器对于ipv4和ipv6都使用同一个socket接口，在同时发出ipv4和ipv6解析请求后，只会收到一个ipv4的解析响应，此时socket将一处于“等待”模式，等待ipv6的解析响应，故导致解析缓慢；添加single-request-reopen后就可以重新打开一个新的socket接收ipv6的解析响应，而不影响ipv4的解析响应。

所以，在容器中修改resolv.conf文件，添加如下：
# vim /etc/resolv.conf
options single-request-reopen
```