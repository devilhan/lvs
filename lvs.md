传说中的三高：
    高并发，高性能，高可用

集群类型：

    scale on : 向上扩展
        将服务器的内存容量调大和cpu数量增加些（升级服务器硬件）
        缺点：在一定的范围之内它的性能是上升的趋势，但是超出范围之后就是下降的趋势。
            因为随着它的cpu的个数增加我们需要给我们的cpu仲裁，而且随着cpu个数的增加，资源竞争性增大。
    
    scale out : 向外扩展
        一台服务器应付不过来，我们就增加一台服务器。
        优点：增减服务器很方便，而且没有向上扩展随着增加性能下降。
        工作模式：当客户端向服务器端发送请求时，服务器根据一定的算法选择一台来响应客户的请求。

LB：Load Balance：负载均衡集群
    负载均衡集群中有一个分发器或者叫作调度器，我们将其称为Director，它处在多台服务器的上面，
    分发器根据内部锁定义的规则或者调度方式从下面的服务集群中选择一台来响应客户端发来的请求。
    负载均衡是对后端服务器的均衡。

HA：High Availability 高可用集群
    高可用集群时服务的可用性比较高，当我们某台服务器死机后不会造成服务不可用。
    其工作模式则是将一个具有故障的服务器的服务转交给一个正常工作的服务器，从而达到服务不会中断。
    一般来说我们集群中工作在前端（分发器）的服务器都会对我们的后端服务器做一个健康检查，如果发现我们服务器down机就不会对其再做转发。
    高可用是对服务器做备用机。
    衡量标准：可用性=在线时间/（在线时间+故障处理时间）99%， 99.9%， 99.99%，99.999%

HP：High Performance 高性能
    这种处理方式我们成为并行处理集群，并行处理集群是将大任务划分为小任务，分别进行处理。
    一般这样的集群用来科学研究与大数据运算等方面的工作。现在比较火的Hadoop就是使用的并行处理集群。

    说明：三种集群之间区别：
        负载均衡着重于提供服务并发处理能力的集群，是对后端服务器的均衡；
        高可用以提升服务在线的能力的集群，是对服务器做了备用机的机制；
        高性能着重于处理一个海量任务，是将一个任务分发给多个主机分别完成。

负载均衡架构：lvs + keepalive

LVS集群采用三层结构，其主要组成部分为：
    A、负载调度器（Load Balancer），它是前端机，负责转发客户的请求，而客户认为服务是来自同一个IP地址。
    B、服务器池（Server Pool），是一组真正执行客户请求的服务器，执行的服务有WEB、MAIL、FTP和DNS等。
    C、共享存储（Shared Storage），它为服务器池提供一个共享的存储区，使服务器池拥有相同内容，提供相同服务。

LVS模型：
    1. 当客户端的请求到达负载均衡器的内核空间时，首先会到达PREROUTING链
    2. 当内核发现数据包的请求目标地址是本机时，将数据包送往INPUT链
    3. lvs是由用户空间的ipvsadm和内核空间的IPVS组成，ipvs用来定义规则，IPVS利用ipvsadm定义的规则工作。
        IPVS工作在INPUT链上，当数据包到达INPUT链时，首先会被IPVS检查，如果数据包里面的ip地址和目标端口没有在ipvsadm定义的规则里，
        那这条数据包将会被放行至用户空间。
    4. 如果数据包里面的ip地址和目标端口在规则里，那么这条数据报文将会送往内核空间的POSTROUTING链，
        接收数据包后发现目标地址正好是自己的后端服务器，那么此时通过选路，将数据包最终送往后端的服务器。

LVS组成
    ipvsadm：用于管理集群服务的命令行工具，工作于linux系统中的用户空间。
    ipvs：为lvs提供服务的内核模块，工作于内核空间（相对于是框架，通过ipvsdm添加规则，来实现ipvs功能）。

    注：在linux内核2.4.23之前的内核模块中ipvs模块是不存在的，需要自己手动打补丁，然后把此模块编译进内核才可以使用此功能。
        rhel5 /rhel6 自带 LVS 软件包 安装 ipvsadm 软件包即可使用。

LVS中每个主机ip地址的定义
    VIP：Director用来向客户端提供服务的IP地址，也是DNS解析的IP
    RIP：集群节点所使用的IP地址（real server）
    DIP: Director用来和RIP进行交互的IP地址
    CIP：公网IP，客户端使用的IP

LVS的包转发模型
    
NAT
    通过修改请求报文的目标IP地址（同时可能修改目标端口，支持端口映射），改为某Real Server的IP地址实现数据包的转发。  
   
    1）客户端将请求报文发往前端的负载均衡器，请求报文源地址为CIP目标地址为VIP
    2）负载均衡器接受到报文，发现请求的是在ipvs规则里面存在的地址，那么它将客户端的请求报文的目标地址改为了后端服务器的RIP地址并将报文根据算法发送出去。
    3）报文送到Real Server上，由于报文的目标地址是自己，所以会响应请求，并将响应报文返还给Director。
    4）然后Director将此报文的源地址修改为本机ip并发送给客户端。
    特点：
        1）集群中各节点跟Directory必须在同一网段
        2）DIP，RIP通常为私有地址，仅用于集群，且Real Server的网关要指向DIP
        3）支持端口映射和转发
        4）Real Server可以使用任意的OS
        5）请求报文和响应报文都要经由Director，较大规模应用场景中Director可能成为系统瓶颈

DR
    1）客户端将请求发往前端的负载均衡器，请求报文源地址是CIP，目标地址为VIP
    2）负载均衡器接收到报文后，发现请求的是在ipvs规则中存在的地址和端口，那么它将客户端请求报文的源MAC地址改为自己的MAC地址，目标MAC改为了Real Server的MAC地址，并将此包发送给Real Server
    3）Real Server发现请求报文中的目标MAC地址是自己，就会把此报文接受下来，处理完请求报文后，将响应报文通过lo接口送给eth0网卡，直接发送给客户端。
    注意：各real server的lo接口上配置的VIP不能响应外部请求。
    
    特点：
    1）集群节点跟Director必须在同一物理网络中
    2）RIP可以使用公网地址，使用便捷的远程控制服务器
    3）Direcotr只负责处理入站请求，响应报文由real server直接发往客户端
    4）real server不能将网关指向DIP
    5）Director不支持端口映射
    6）real server支持应用在大多数OS
    7）DR比NAT能处理更多的real server

    VIP地址为调度器和服务器组共享,调度器配置的VIP地址是对外可见的,用于接收虚拟服务的请求报文;
    所有的服务器把VIP地址配置在各自的Non-ARP网络设备上,它对外面是不可见的,只是用于处理目标地址为VIP 的网络请求
    
TUN
    1）客户端将请求报文法网前端的负载均衡器，请求报文源地址是CIP，目标地址为VIP
    2）负载均衡器受到报文后，发现请求的是在IPVS规则中存在的地址和对应的端口，那么它将在客户端的请求报文的首部再封装一层IP报文，源地址为DIP，目标地址为RIP，并将此包发送给RS。
    3）RS收到请求报文后，会首先拆开第一层封装，然后发现里面还有一层IP首部的目标地址是自己lo接口上的VIP，所以会再次处理请求报文，并将响应报文通过lo接口送往eth0网卡直接发送给客户端。
    注意：需要设置lo接口上的VIP不能出现在公网上。
    
    特点：
    1）各集群节点可以跨越不同的网络
    2）RIP，DIP，VIP必须是公网地址
    3）DIrector只负责处理入站请求，响应报文由real server直接发往客户端
    4）real server网关不能指向Director
    5）real server仅能搭建在支持隧道功能的主机上
    6）不支持端口映射





