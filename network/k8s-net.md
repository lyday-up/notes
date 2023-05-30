### linux network

linux网络，本质上就是由一系列组件组成，从而共同协作完成网络功能，一般这些组件包括：

- linux网络设备：例如network interface device，loop back device，bridge device，veth device，tun/tap device，vxlan device，ip tunnel device等等。这些设备可以完成网络数据包的收发，以及提供额外的修改数据包等功能。
  
- linux路由表，arp表，fdb(forwarding database)等：路由表提供三层ip包的路由寻址功能，arp表提供ip对应的二层mac地址，fdb在在基于mac转发的功能里提供mac地址对应的网络接口。
  
- linux协议栈：完成对网络协议包的封装与解析，例如二层ethernet包，三层ip包，四层tcp/udp包等等。
  
- linux iptable：iptable基于linux内核模块netfilter，完成对于linux的firewall管理，例如控制ingress与engress，nat地址转换，端口映射等等。
  

### linux network namespace

linux network namespace就像是一个可以相互隔离的组一样，把网络设备，路由表，协议栈，iptable等组件包装起来，然后通过namespace来相互隔离，互不干扰。所以更具体的说，网络设备，路由表，协议栈，iptable等是工作在某一个linux network namespace下的

linux不仅仅只有network namespace用来进行网络隔离，还有pid namespace用来隔离进程，user namespace用来隔离用户，mount namespace用来隔离挂载点，ipc namespace用来隔离信号量和共享内存等，uts namespace用来隔离主机名和域名。

### Linux Bridge网桥

Bridge是一种linux网络设备，可以附加attach多个linux从设备。可以把linux bridge想象成二层交换机，可以进行二层数据包的广播，但是注意的是linux bridge设备可以有自己的ip地址。多个linux网络设备attach到一个bridge上，那么这些网络设备的ip地址将会失效(只有二层功能)，当一个设备收到数据包的时候，bridge会把数据包转发到其它所有attach到bridge上的从设备，从而实现广播的效果。

### Linux Veth设备

Veth设备总是成对出现，它的特点是会有一对peer，两个端点，数据包从一个peer流入，总是可以从另一个peer流出。而且veth pair是可以跨linux network namespace的。因为network namespace本质就是相互隔离的，可以用veth来做到数据包的跨network namespace的访问。一个peer在一个namespace里，另一个peer在另一个namespace里。

### linux TUN设备：

- linux TUN device：Linux TUN device是一种网络设备，它可以有自己的ip地址，其最重要的特性就是可以被应用程序监听读写，被监听读写的对象为三层ip数据包。
  
- TUN device一端连接网络内核空间，另一端连接用户空间的应用程序。
  
- 用户空间的应用程序有能力通过TUN device读写修改三层ip数据包。
  
- 数据从内核空间进入TUN device的时候，该设备会把数据交给用户空间的应用程序，使得应用程序有机会根据自己的逻辑修改ip数据包。
  
- 用户空间的应用程序可以把数据发给TUN device，该设备会把数据交由内核空间进行路由转发。
  
- flannel udp模式就是利用TUN device，由flannel进程完成对原始ip包的udp封包，然后转发并解包。
  

### docker network

对于docker宿主机环境中容器的网络一般是：

- 每一个container都有一个network namespace，宿主中容器的网络地址空间一般为x.x.x.0/24，然后拥有container自己的网络设备，路由表，arp表，协议栈，iptable等，各个container的network namespace相互隔离。
  
- 在宿主的default netwok nemespace中会有一个linux bridge设备，一般名称为docker0。container的默认网关是x.x.x.1/32，也就是宿主network namespace中的linux bridge docker0的ip地址。
  
- 每一个container对应一个veth pair设备，这个设备的一端在container的network namespace里，另一端attach到宿主networkwork namespace的docker0 linux bridge上。
  
- 每个container会有一个veth pair device对应，这个veth pair一端在container的network namespace中，有ip地址，并且这个ip地址就是容器的ip地址。veth pair的另一端attach到宿主network namespace中的linux bridge docker0上，以完成container network namespace和宿主network namespace之间的数据流动。
  
- 这样在宿主环境里，就好像有一个二层交换机(docker0 bridge)，把宿主内的所有container连接起来。所以，在宿主内的container都是可以直接相互访问的，而且是直连的方式。
  
- container到容器的网络地址空间(x.x.x.0/24)的访问方式为直连，不需要下一跳ip。
  

### k8s service

cluster ip是虚拟ip，就是这个ip没有和任何device绑定，所以当你对这个ip进行例如ping或者traceroute命令的时候是不会得到应答的。

ipable方式的k8s集群内cluster-ip类型的service总结为：

- 流量从pod network namespace中走到host netwok namespace的docker0中。
  
- 在host netwok namespace的PREROUTING chain中会经过一系列target。
  
- 在这些target里根据iptable内核随机模块来实现匹配endpoint target，随机比率为均匀分配，实现均匀的负载均衡。
  
- 在endpoint target里实现了DNAT，也就是将目标地址cluster ip转化为实际的pod的ip。
  
- cluster ip是虚拟ip，不会和任何device绑定。
  
- 负载均衡为内核实现，使用均匀负载均衡，不可以有自定义的负载均衡算法。
  
- 需要host开启路由转发功能(net.ipv4.ip_forward = 1)。
  
- 数据包在host netwok namespace中经过转换以及DNAT之后，由host network namespace的路由表来决定下一跳地址。
  

ipable方式的k8s集群内node port类型的service总结为：

- 在host netwok namespace的PREROUTING chain中会匹配KUBE-SERVICES target。
  
- 在KUBE-SERVICES target会匹配KUBE-NODEPORTS target
  
- 在KUBE-NODEPORTS target会根据prot来匹配KUBE-SVC-XXX target
  
- KUBE-SVC-XXX target就和cluster-ip类型service一样了
  

ipvs下的cluster ip的通讯方式为：

- 数据包从pod network namespace发出，进入host的network namespace，源ip为pod ip，源端口为随机端口，目标ip为cluster ip，目标port为指定port。
  
- 数据包在host network namespace中进入PREROUTING chain。
  
- 在PREROUTING chain中经过匹配ipset KUBE-CLUSTER-IP做mask标记操作。
  
- 在host network namespace中创建网络设备kube-ipvs0，并且绑定所有cluster ip，这样从pod发出的数据包目标ip为cluster ip，有kube-ipvs0网络设备对应，数据进入INPUT chain中。
  
- 数据在INPUT chain中被ipvs的内核规则修改(可由ipvsadm查看规则)，完成DNAT，然后将数据直接送入POSTROUTING chain。这时源ip为pod ip，源端口为随机端口，目标ip为映射选择的pod ip，目标port为映射选择的port。
  
- 数据在POSTROUTING chain中，经过KUBE-POSTROUTING target完成MASQUERADE SNAT。这时源ip为下一跳路由所使用网路设备的ip，源端口为随机端口，目标ip为映射选择的pod ip，目标port为映射选择的port。
  
- 数据包根据host network namespace的路由表做下一跳路由选择。
  

ipvs下的NodePort ip的通讯方式为：

- 数据包从host外部访问node port service的host端口，进入host的network namespace，源ip为外部 ip，源端口为随机端口，目标ip为host ip，目标port为node port。
  
- 数据包在host network namespace中进入PREROUTING chain。
  
- 在PREROUTING chain中经过匹配ipset KUBE-NODE-PORT-TCP做mask标记操作。
  
- 因为数据包的目标ip是host的ip地址，所以进入host network namespace的INPUT chain中。
  
- 数据在INPUT chain中被ipvs的内核规则修改(可由ipvsadm查看规则)，完成负载均衡和DNAT，然后将数据直接送入POSTROUTING chain。这时源ip为外部ip，源端口为随机端口，目标ip为映射选择的pod ip，目标port为映射选择的port。
  
- 数据在POSTROUTING chain中，经过KUBE-POSTROUTING target完成MASQUERADE SNAT。这时源ip为下一跳路由所使用网路设备的ip，源端口为随机端口，目标ip为映射选择的pod ip，目标port为映射选择的port。
  
- 数据包根据host network namespace的路由表做下一跳路由选择。
  

###

### 对于iptable方式的service：

- 流量从pod network namespace（cluster ip类型的service）或者外部(node port类型的service)进入到host netwok namespace之中。
  
- 在host netwok namespace的PREROUTING chain中会经过一系列target，KUBE-SERVICES(cluster ip类型的service)，KUBE-NODEPORTS (node port类型的service)，KUBE-SVC-XXX，KUBE-SEP-XXX。
  
- 在这些target里根据iptable内核随机模块random来实现匹配endpoint target，实现负载均衡。
  
- 在endpoint target(KUBE-SEP-XXX)里实现了DNAT，也就是将目标地址cluster ip转化为实际的pod的ip。
  
- 数据包经过以上修改根据host network namespace的路由表做下一跳路由选择。
  

### 对于ipvs方式的service：

- 流量从pod network namespace（cluster ip类型的service）或者外部(node port类型的service)进入到host netwok namespace之中。
  
- 对于clutser ip类型的service，在host netwok namespace的PREROUTING chain中经过匹配ipset KUBE-CLUSTER-IP做mask标记操作。
  
- 对于node port类型的service，在PREROUTING chain中经过匹配ipset KUBE-NODE-PORT-TCP做mask标记操作。
  
- 对于clutser ip类型的service，由于host network namespace中有创建网络设备kube-ipvs0，并且绑定所有cluster ip，这样从pod发出的数据包目标ip为cluster ip，有kube-ipvs0网络设备对应，数据进入INPUT chain中。
  
- 对于node port类型的service，由于数据包的目标ip是host的ip地址，所以也进入了host network namespace的INPUT chain中。
  
- 利用linux内核模块ipvs，数据在INPUT chain中被ipvs的规则修改(可由ipvsadm查看规则)，完成负载均衡和DNAT，然后将数据直接送入POSTROUTING chain。
  
- 数据在POSTROUTING chain中，经过KUBE-POSTROUTING target，根据之前的mark操作完成MASQUERADE SNAT。
  
- 数据包经过以上修改根据host network namespace的路由表做下一跳路由选择。
  

### iptable和ipvs方式的service：

- 两者都是采用linux内核模块完成负载均衡和endpoint的映射，所有操作都在内核空间完成，没有在应用程序的用户空间。
  
- iptable方式依赖于linux netfilter/iptable内核模块。
  
- ipvs方式依赖linux netfilter/iptable模块，ipset模块，ipvs模块。
  
- iptable方式中，host宿主中ipatble的entry数目会随着service和对应endpoints的数目增多而增多。举个例子，比如有10个cluster ip类型的service，每个service有6个endpoints。那么在KUBE-SERVICES target中至少有10个entries(KUBE-SVC-XXX)与10个service对应，每个KUBE-SVC-XXX target中会有6个KUBE-SEP-XXX与6个endpoints来对应，每个KUBE-SEP-XXX会有2个enrties来分别做mark masq和DNAT，这样算起来至少有10*6*2=120个entries在iptable中。试想如果application中service和endpoints数目巨大，iptable entries也是非常庞大的，在一定情况下有可能带来性能上的问题。
  
- ipvs方式中host宿主中iptable的entry数目是固定的，因为iptable做匹配的时候会利用ipset(KUBE-CLUSTER-IP或者KUBE-NODE-PORT-TCP)来匹配，service的数目决定了ipset的大小，并不会影响iptable的大小。这样就解决了iptable模式下，entries随着service和endpoints的增多而增多的问题。
  
- 对于负载均衡，iptable方式采用random模块来完成负载均衡，ipvs方式支持多种负载均衡，例如round-robin，least connection，source hash等（可参考http://www.linuxvirtualserver.org/），并且由kubelet启动参数--ipvs-scheduler控制。
  
- 对于目标地址的映射，iptable方式采用linux原生的DNAT，ipvs方式则利用ipvs模块完成。
  
- ipvs方式会在host netwok namespace中创建网络设备kube-ipvs0，并且绑定了所有的cluster ip，这样保证了cluster-ip类型的service数据进入INPUT chain，从而让ipvs来完成负载均衡和目标地址的映射。
  
- iptable方式不会在host netwok namespace中创建额外的网络设备。
  
- iptable方式数据在host network namespace的chain中的路径是：PREROUTING-->FORWARDING-->POSTROUTING 
  
  在PREROUTING chain中完成负载均衡，mark masq和目标地址映射。
  
- ipvs方式数据在host network namespace的chain中的路径是：
  
  PREROUTING-->INPUT-->POSTROUTING 
  
  在PREROUTING chain中完成mark masq SNAT，在INPUT chain利用ipvs完成负载均衡和目标地址映射。iptable和ipvs方式在完成负载均衡和目标地址映射后都会根据host network namespace的路由表做下一跳路由选择。
  

### 容器网络通信

对于容器之间的网络通讯总结起来基本分为两种，underlay方式和overlay方式。underlay方式在通讯过程中没有额外的封包，通常将容器的宿主作为路由来实现数据包的转发，需要宿主开启网络转发功能(net.ipv4.ip_forward = 1)。常见的实现方案有flannel host-gw方式，calico bgp方式

overlay方式在通讯过程中有额外的封包，例如flannel vxlan方式(在三层网络里构建二层网络，即在udp包里封装eth以太包)，calico ipip模式(在ip包里再次封装ip包)。还有flannel udp方式，在upd包里封装ip包

### flannel host gw

flannel的underlay网络host gw方式的限制，既要求所有的k8s worker node节点都在同一个二层网络里(也可以认为是在同一个ip子网里)

目标pod的下一跳地址是目标pod所在的host，也就是说数据会从原始pod所在的host通过下一跳发往目标pod所在的host

underlay(flannel host gw方式)，数据包：

- 从源pod的network namespace到host network namespace的docker0 linux bridge上。
  
- 在源pod所在的host里做三层路由选择，下一跳地址为目标pod所在的host。
  
- 数据包从源pod所在的host发送到目标pod所在的host。
  
- 在目标pod所在的host里做三层路由选择，本地直连路由到目标pod里。
  
- 要求所有的worker node必须开启路由转发功能(net.ipv4.ip_forward = 1)。
  
- 要求所有的worker node都在同一个二层网络里，来完成目标pod所在host的下一跳路由。
  

### flannel vxlan overlay

所有的host都运行flannel服务，而flannel连接etcd存储中心，所以每个host就知道自己的子网地址cidr是什么，也知道在这个cidr中自己的flannel.1设备ip地址和mac地址，同时也知道了其它host的子网cidr以及flannel.1设备ip地址和mac地址。而知道了这些信息，就可以在flannel启动的时候写入到路由表和fdb中了。

flannel vxlan overlay网络pod到pod的通讯过程如下：

- 每个宿主都有名字为flannel.x的vxlan网络设备来完成对于vxlan数据的udp封包与拆包。
  
- 数据从pod的network namespace进入到host的network namespace中。
  
- 根据host network namespace中的路由表，下一跳ip为目标vxlan设备的ip，并且由当前host的flannel.x设备发送。
  
- 根据host network namespace中的apr表找到下一跳ip的mac地址。
  
- 根据host network namespace中fbd找到下一跳ip的mac地址对应的转发ip。
  
- 当前host的flannel.x设备根据下一跳ip的mac地址对应的转发ip和本地路由表进行upd封包。
  
- 外层udp包：源ip为当前host ip，目标ip为mac转发表中匹配的ip，源mac为前host ip的mac，目标mac为fdb中匹配ip的mac。目标端口为8472(可配置)，vxlan id为1(可配置).
  
- 内层二层以太包：源ip为源pod ip，目标ip为目标pod ip，源mac为源pod mac，目标mac为host network namespace中路由表里下一跳ip的mac(一般为目标pod对应的host中flannel.x设备ip)。
  
- 数据包由当前host路由到目标节点host。
  
- 目标节点host的8472端口接收到udp包之后，发现数据包里有vxlan id标识.。然后根据linux vxlan协议，在目标宿主机器上找到与数据报文中vxlan id对应的vxlan设备，将数据交由其处理。
  
- vxlan设备收到数据之后开始对vxlan udp报文拆包，去掉upd报文的ip，port，mac信息后得到内部的payload，发现是一个二层报文。然后继续对这个二层报文拆包，得到里面的源pod ip和目标pod ip。
  
- 根据目标节点host上路由表，将数据由linux bridge docker0做本地转发。
  
- 数据由linux bridge docker0利用veth pair转发到目标pod。
  
- 每个宿主host的flannel服务启动的时候读取etcd中的vxlan配置信息，在宿主host的路由表和mac转发接口表fdb里写入相应数据。
  
  ![](https://static001.geekbang.org/resource/image/03/f5/03185fab251a833fef7ed6665d5049f5.jpg?wh=1767*933)
  

### flannel udp overlay

flannel udp overlay网络pod到pod的通讯过程如下：

- 每个宿主都有名字为flannel0的TUN网络设备来完成对于原始ip数据包的udp封包与拆包，upd数据在宿主的8285端口上(端口值可配置)的flannel进程处理。
  
- 数据从原始pod的network namespace进入到host的network namespace中。
  
- 根据host network namespace中的路由表，下一跳ip为目标为直连，并且由当前host的TUN设备flannel0发送。
  
- 当前host的TUN设备flannel0将数据从内核空间交给用户空间应用程序flannel。
  
- 户空间应用程序flannel根据目标ip属于哪个子网，然后在etcd中查询这个子网所在的目标host，最后完成upd封包，这个时候：
  
- 对于原始三层包：源ip为源pod ip，目标ip为目标pod ip。
  
- 对于外层udp包：源ip为当前host的ip，目标ip为flannel在etcd中查询的匹配ip，目标port为8285(可以配置)，源端口为随机端口。
  
- flannel将upd包发送给TUN设备flannel0，数据由用户空间进入内核空间。
  
- 数据在内核空间根据路由策略发送到目标宿主的8285端口。
  
- 目标宿主的8285端口为flannel进程，数据由内核空间进入用户空间。
  
- flannel进程对udp数据拆包，去掉ip和port信息，得到内层的原始ip包。然后把这个包发送给TUN device flannel0，数据由应用程序用户空间进入内核空间。
  
- 根据目标节点host上路由表，将数据由linux bridge docker0做本地转发。
  
- 数据由linux bridge docker0利用veth pair转发到目标pod。
  

![](https://static001.geekbang.org/resource/image/83/6c/8332564c0547bf46d1fbba2a1e0e166c.jpg?wh=1857*878)

对于flannel underlay网络：

- 数据包从源pod的network namespace到host network namespace的docker0 linux bridge上。
  
- 在源pod的host里做路由选择，下一跳地址为目标pod的host。
  
- 在目标pod的host里做路由选择，本地直连路由到目标pod里。
  
- 要求所有的worker node必须开启路由转发功能(net.ipv4.ip_forward = 1)。
  
- 要求所有的worker node都在同一个二层网络里，来完成目标pod的下一跳路由。
  
- host里的路由会在flannel启动的时候写入，因为host连接etcd集群，知道哪个子网属于哪个host。
  

对于flannel vxlan overlay网络：

- 每个宿主都有名字为flannel.x的vxlan网络设备来完成对于vxlan数据的udp封包与拆包，upd数据在宿主的8472端口上(端口值可配置)处理。
  
- 数据从pod的network namespace进入到host的network namespace中。
  
- 根据host network namespace中的路由表，下一跳ip为目标vxlan设备的ip，并且由当前host的flannel.x设备发送。
  
- 根据host network namespace中的apr表找到下一跳ip的mac地址。
  
- 根据host network namespace中fbd转发表找到下一跳ip的mac地址对应的转发ip。
  
- 当前host的flannel.x设备根据下一跳ip的mac地址对应的转发ip和本地路由表进行upd封包，这个时候：
  
- 外层udp包：源ip为当前host ip，目标ip为mac转发表中匹配的ip，源mac为前host ip的mac，目标mac为fdb中匹配ip的mac。目标端口为8472(可配置)，vxlan id为1(可配置).
  
- 内层二层以太包：源ip为源pod ip，目标ip为目标pod ip，源mac为源pod mac，目标mac为host network namespace中路由表里下一跳ip的mac(一般为目标pod对应的host中flannel.x设备ip的mac)。
  
- 数据包由当前host路由到目标节点host。
  
- 目标节点host的8472端口接收到udp包之后，发现数据包里有vxlan id标识.。然后根据linux vxlan协议，在目标宿主机器上找到与数据报文中vxlan id对应的vxlan设备，将数据交由其处理。
  
- vxlan设备收到数据之后开始对vxlan udp报文拆包，去掉upd报文的ip，port，mac信息后得到内部的payload，发现是一个二层报文。然后继续对这个二层报文拆包，得到里面的源pod ip和目标pod ip。
  
- 根据目标节点host上路由表，将数据由linux bridge docker0做本地转发。
  
- 数据由linux bridge docker0利用veth pair转发到目标pod。
  
- 每个host的flannel服务启动的时候读取etcd中的vxlan配置信息，在host的路由表的和mac转发接口表fdb里写入相应数据。
  

对于flannel udp overlay网络：

- 每个宿主都有名字为flannel0的TUN网络设备来完成对于原始ip数据包的udp封包与拆包，upd数据在宿主的8285端口上(端口值可配置)的flannel进程处理。
  
- 数据从pod的network namespace进入到host的network namespace中。
  
- 根据host network namespace中的路由表，下一跳ip为目标为直连，并且由当前host的TUN设备flannel0发送。
  
- 当前host的TUN设备flannel0将数据从内核空间交给用户空间应用程序flannel。
  
- 户空间应用程序flannel根据目标ip属于哪个子网，然后在etcd中查询这个子网所在的目标host，最后完成upd封包，这个时候：
  
- 对于原始ip包：源ip为源pod ip，目标ip为目标pod ip。
  
- 对于外层udp包：源ip为当前host的ip，目标ip为flannel在etcd中查询的匹配ip，目标port为8285(可以配置)，源端口为随机端口。
  
- flannel将upd包发送给TUN设备flannel0，数据由用户空间进入内核空间。
  
- 数据在内核空间根据路由策略发送到目标宿主的8285端口。
  
- 目标宿主的8285端口为flannel进程，数据由内核空间进入用户空间。
  
- flannel进程对udp数据拆包，去掉ip和port信息，得到内层的原始ip包。然后把这个包发送给TUN device flannel0，数据由程序用户空间进入内核空间。
  
- 根据目标节点host上路由表，将数据由linux bridge docker0做本地转发。
  
- 数据由linux bridge docker0利用veth pair转发到目标pod。
  

对于以上三种方式：

- 都要求host宿主开启网络转发功能(net.ipv4.ip_forward = 1)。
  
- flannel underlay网络没有数据包的额外封包与拆包，效率会更高一些。
  
- 对于flannel underlay网络要求所有的worker node都在同一个二层网络里，从而完成目标pod的下一跳路由。换句话说，underlay网络worker node不能跨子网。
  
- flannel vxlan overlay网络和flannel udp overlay网络的worker node只要三层路由可达就好，支持worker node能跨子网。
  
- 对于flannel vxlan overlay网络和flannel udp overlay网络都有封包与拆包，并且外层包都是udp包。
  
- flannel vxlan overlay网络内层包是二层以太包，flannel udp overlay网络内层包是三层ip包。
  
- flannel vxlan overlay网络基于linux vxlan设备，flannel udp overlay网络基于linux TUN设备。
  
- flannel underlay网络和flannel vxlan overlay网络所有数据包都由操作系统内核空间处理，没有用户空间的应用程序参与。
  
- flannel udp overlay网络数据包既有操作系统内核空间处理，也有用户空间的应用程序flannel参与，所以udp overlay网络效率最低，基本不会在生产上使用。
