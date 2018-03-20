---
layout:     post
title:      "负载均衡原理"
subtitle:   ""
date:       2018-01-18 12:00:00
author:     "Tango"
header-img: "img/in-post/post-2017-10-11/post-bg-universe.jpg"
catalog: true
tags:   
    - 系统架构
    - 负载均衡
---


>最近在学习负载均衡原理，看了大量的文章和相关知识，有很大的收获，做一个简要总结，一来回顾自己所学的知识，二来对所看所学知识整理，供后来者参考借鉴，如有错误的地方，还请指出，学生阶段，考虑的内容尚有欠缺之处。

### 1.什么是负载均衡

负载均衡，英文名称为Load Balance，指由多台服务器以对称的方式组成一个服务器集合，每台服务器都具有等价的地位，都可以单独对外提供服务而无须其他服务器的辅助。通过某种负载分担技术，将外部发送来的请求均匀分配到对称结构中的某一台服务器上，而接收到请求的服务器独立地回应客户的请求。负载均衡能够平均分配客户请求到服务器阵列，借此提供快速获取重要数据，解决大量并发访问服务问题，这种集群技术可以用最少的投资获得接近于大型主机的性能。

![](/img/in-post/post-load-balance.png)

负载均衡系统架构如上图所示，其实际是代理模式的实现。来自客户端的请求经由负载均衡器，按照某种均衡算法分配到后端服务器进行响应。后端服务器处理后，将响应报文返回负载均衡服务器，负载均衡服务器再返回客户端。

负载均衡在服务端开发中算是一个比较重要的特性。负载均衡分为：

- 硬件负载均衡

  `专用的软件和硬件相结合的设备，设备商会提供完整成熟的解决方案，通常也会更加昂贵，例如：思杰。`

- 软件负载均衡

 - 四层负载均衡(基于IP地址转发)  
 
	`四层负载均衡工作在OSI模型的传输层，主要工作是转发，它在接收到客户端的流量以后通过修改数据包的地址信息将流量转发到应用服务器。`    
	
 - 七层负载均衡(基于应用层协议转发)

	`七层负载均衡工作在OSI模型的应用层，因为它需要解析应用层流量，所以七层负载均衡在接到客户端的流量以后，还需要一个完整的TCP/IP协议栈。
七层负载均衡会与客户端建立一条完整的连接并将应用层的请求流量解析出来，再按照调度算法选择一个应用服务器，并与应用服务器建立另外一条连接将请求发送过去，因此七层负载均衡的主要工作就是代理。`
	
既然四层负载均衡做的主要工作是转发，那就存在一个转发模式的问题，目前主要有四层转发模式：DR模式、NAT模式、TUNNEL模式。三种模式将在下文详细介绍。

### 2.IPVS实现机制

1) **VS/NAT：即（VirtualServer via Network Address Translation）**. 

```
网络地址翻译技术实现虚拟服务器，当用户请求到达调度器时，调度器将请求报文的目标地址（即虚拟IP地址）改写成选定的RealServer地址，
同时报文的目标端口也改成选定的RealServer的相应端口，最后将报文请求发送到选定的RealServer。
在服务器端得到数据后，RealServer返回数据给用户时，需要再次经过负载调度器将报文的源地址和源端口改成虚拟IP地址和相应端口，然后把数据发送给用户，完成整个负载调度过程。
可以看出，在NAT方式下，用户请求和响应报文都必须经过DirectorServer地址重写，当用户请求越来越多时，调度器的处理能力将称为瓶颈。

优点：集群中的物理服务器可以使用任何支持TCP/IP操作系统，物理服务器可以分配Internet的保留私有地址，只有负载均衡器需要一个合法的IP地址。

缺点：扩展性有限。当服务器节点（普通PC服务器）数据增长到20个或更多时,负载均衡器将成为整个系统的瓶颈，因为所有的请求包和应答包都需要经过负载均衡器再生。假使TCP包的平均长度是536字节的话，平均包再生延迟时间大约为60us（在Pentium处理器上计算的，采用更快的处理器将使得这个延迟时间变短），负载均衡器的最大容许能力为93M/s，假定每台物理服务器的平台容许能力为400K/s来计算，负责均衡器能为22台物理服务器计算。
```

2)**VS/TUN :即（Virtual Server via IP Tunneling）**. 

```
IP隧道技术实现虚拟服务器。它的连接调度和管理与VS/NAT方式一样，只是它的报文转发方法不同，
VS/TUN方式中，调度器采用IP隧道技术将用户请求转发到某个RealServer，而这个RealServer将直接响应用户的请求，不再经过前端调度器.
此外，对RealServer的地域位置没有要求，可以和DirectorServer位于同一个网段，也可以是独立的一个网络。因此，在TUN方式中，调度器将只处理用户的报文请求，集群系统的吞吐量大大提高。

优点：负载均衡器只负责将请求包分发给物理服务器，而物理服务器将应答包直接发给用户。
所以，负载均衡器能处理很巨大的请求量，这种方式，一台负载均衡能为超过100台的物理服务器服务，负载均衡器不再是系统的瓶颈。使用VS-TUN方式，如果你的负载均衡器拥有100M的全双工网卡的话，就能使得整个VirtualServer能达到1G的吞吐量。

缺点：这种方式需要所有的服务器支持”IP Tunneling”(IP Encapsulation)协议，仅在Linux系统上实现了。
```
3) **VS/DR：即（VirtualServer via Direct Routing）**. 

```
即用直接路由技术实现虚拟服务器。它的连接调度和管理与VS/NAT和VS/TUN中的一样，
但它的报文转发方法又有不同，VS/DR通过改写请求报文的MAC地址，将请求发送到RealServer，而RealServer将响应直接返回给客户，免去了VS/TUN中的IP隧道开销。
这种方式是三种负载调度机制中性能最高最好的，但是必须要求DirectorServer与RealServer都有一块网卡连在同一物理网段上。

优点：和VS－TUN一样，负载均衡器也只是分发请求，应答包通过单独的路由方法返回给客户端。
与VS-TUN相比，VS-DR这种实现方式不需要隧道结构，因此可以使用大多数操作系统做为物理服务器，

缺点：要求负载均衡器的网卡必须与物理网卡在一个物理段上
```

### 3.负载均衡策略

(1)**HTTP重定向策略**  

```
当用户发来请求的时候，Web服务器通过修改HTTP响应头中的Location标记来返回一个新url，然后浏览器再继续请求这个新url，实际上就是页面重定向。
通过重定向，来达到“负载均衡”的目标这个方式非常容易实现，并且可以自定义各种策略，但是，它在大规模访问量下，性能不佳，而且，给用户的体验也不好，实际请求发生重定向，增加了网络延时所以此方式了解即可，实际应用较少
``` 

(2) **反向代理策略**. 

```
反向代理服务的核心工作主要是转发HTTP请求，扮演了浏览器端和后台Web服务器中转的角色。因为它工作在HTTP层（应用层），也就是网络七层结构中的第七层，因此也被称为“七层负载均衡”。
可以做反向代理的软件很多，比较常见的一种是Nginx，Nginx是一种非常灵活的反向代理软件，可以自由定制化转发策略，分配服务器流量的权重等

优点：实现和部署非常简单，性能也很好，可以方便的自定义转发规则

缺点：有“单点故障”的问题，如果挂了，会带来很多的麻烦，而且，随着Web服务器继续增加，它本身可能成为系统的瓶颈
```
**(3)IP负载均衡策略**. 

```
原理：它是对IP层的数据包的IP地址和端口信息进行修改，达到负载均衡的目的在负载均衡服务器收到客户端的IP包的时候，会修改IP包的目标IP地址或端口，然后原封不动地投递到内部网络中，数据包会流入到实际Web服务器。
实际服务器处理完成后，又会将数据包投递回给负载均衡服务器，它再修改目标IP地址为用户IP地址，最终回到客户端。因为它工作在网络层，也就是网络七层结构中的第4层，因此也被称为“四层负载均衡”。
常见的负载均衡方式，是LVS（LinuxVirtual Server，Linux虚拟服务），通过IPVS（IP Virtual Server，IP虚拟服务）来实现。

优点：性能比反向代理负载均衡高很多，非常稳定缺点：功能单一，配置复杂
``` 

**(4)DNS负载均衡策略**. 

```
DNS（Domain Name System）负责域名解析的服务，域名url实际上是服务器的别名，实际映射是一个IP地址，解析过程，就是DNS完成域名到IP的映射。而一个域名是可以配置成对应多个IP的。
因此，DNS也就可以作为负载均衡服务优点：配置简单，性能极佳缺点：不能自由定义规则，而且，变更被映射的IP或者机器故障时很麻烦，还存在DNS生效延迟的问题
```

###4.负载均衡算法

**(1)随机法**

通过系统随机函数，根据后端服务器列表的大小值来随机选择其中一台进行访问。由概率统计理论可以得知，随着调用量的增大，其实际效果越来越接近于平均分配流量到每一台后端服务器，也就是轮询的效果。

随机法的代码实现大致如下：

```
public class Random
{
   public static String getServer()
   {
       // 重建一个Map，避免服务器的上下线导致的并发问题
       Map<String, Integer> serverMap = 
               new HashMap<String,Integer>();
       serverMap.putAll(IpMap.serverWeightMap);
        // 取得Ip地址List
       Set<String> keySet = serverMap.keySet();
       ArrayList<String> keyList = newArrayList<String>();
       keyList.addAll(keySet);
        java.util.Randomrandom = new java.util.Random();
       int randomPos = random.nextInt(keyList.size());
        returnkeyList.get(randomPos);
   }
}
```

整体代码思路和轮询法一致，先重建serverMap，再获取到server列表。在选取server的时候，通过Random的nextInt方法取0~keyList.size()区间的一个随机值，从而从服务器列表中随机获取到一台服务器地址进行返回。基于概率统计的理论，吞吐量越大，随机算法的效果越接近于轮询算法的效果。

**(2)轮询(RoundRobin)法**

原理：负载均衡服务器建立一张服务器map,客户端请求到来，按照顺序依次分配给后端服务器处理。  

```
public class RoundRobin
{
   private static Integer pos = 0;
    public static String getServer()
   {
       // 重建一个Map，避免服务器的上下线导致的并发问题
       Map<String, Integer> serverMap = 
               new HashMap<String,Integer>();
       serverMap.putAll(IpMap.serverWeightMap);
 
       // 取得Ip地址List
       Set<String> keySet = serverMap.keySet();
       ArrayList<String> keyList = newArrayList<String>();
       keyList.addAll(keySet);
 
       String server = null;
       synchronized (pos)
       {
           if (pos > keySet.size())
               pos = 0;
           server = keyList.get(pos);
           pos ++;
       }
 
       return server;
   }
}
```  

由于serverWeightMap中的地址列表是动态的，随时可能有机器上线、下线或者宕机，因此为了避免可能出现的并发问题，方法内部要新建局部变量serverMap，现将serverMap中的内容复制到线程本地，以避免被多个线程修改。
这样可能会引入新的问题，复制以后serverWeightMap的修改无法反映给serverMap，也就是说这一轮选择服务器的过程中，新增服务器或者下线服务器，负载均衡算法将无法获知。
新增无所谓，如果有服务器下线或者宕机，那么可能会访问到不存在的地址。因此，服务调用端需要有相应的容错处理，比如重新发起一次server选择并调用。

对于当前轮询的位置变量pos，为了保证服务器选择的顺序性，需要在操作时对其加锁，使得同一时刻只能有一个线程可以修改pos的值，否则当pos变量被并发修改，则无法保证服务器选择的顺序性，甚至有可能导致keyList数组越界。

   优点：试图做到请求转移的绝对均衡。

  缺点：为了做到请求转移的绝对均衡，必须付出相当大的代价，因为为了保证pos变量修改的互斥性，需要引入重量级的悲观锁synchronized，这将会导致该段轮询代码的并发吞吐量发生明显的下降。

**(3)加权轮询法**

轮询法负载均衡基于请求数的合理平衡分配，实现负载均衡。但这存在一种问题：对于后端服务器A、B，A的配置较B服务器较高，高并发到来n个请求，按照轮询法原理，A、B服务器各分配n/2各请求，但由于A服务器配置较高，很快处理完请求，而B服务器并没有处理完所有请求，当再次到来m个请求时，再次分包分配m/2个请求，长此下去，B服务器积聚较多的请求。这其实并没有达到均衡负载的目的，系统处于“伪均衡”状态。

不同的服务器可能机器配置和当前系统的负载并不相同，因此它们的抗压能力也不尽相同，给配置高、负载低的机器配置更高的权重，让其处理更多的请求，而低配置、高负载的机器，则给其分配较低的权重，降低其系统负载。加权轮询法可以很好地处理这一问题，并将请求顺序按照权重分配到后端。

```
public class WeightRoundRobin
{
   private static Integer pos;
 
   public static String getServer()
   {
       // 重建一个Map，避免服务器的上下线导致的并发问题
       Map<String, Integer> serverMap = 
               new HashMap<String,Integer>();
       serverMap.putAll(IpMap.serverWeightMap);
 
       // 取得Ip地址List
       Set<String> keySet = serverMap.keySet();
       Iterator<String> iterator = keySet.iterator();
 
       List<String> serverList = newArrayList<String>();
       while (iterator.hasNext())
       {
           String server = iterator.next();
           int weight = serverMap.get(server);
           for (int i = 0; i < weight; i++)
               serverList.add(server);
       }
 
       String server = null;
       synchronized (pos)
       {
           if (pos > keySet.size())
               pos = 0;
           server = serverList.get(pos);
           pos ++;
       }
        return server;
   }
}  
```
与轮询法类似，只是在获取服务器地址之前增加了一段权重计算的代码，根据权重的大小，将地址重复地增加到服务器地址列表中，权重越大，该服务器每轮所获得的请求数量越多。

**(4)最小连接数**

最小连接数算法基于后端服务器状态(未处理请求队列大小)进行合理分配方式，每次客户端请求到来，负载均衡服务器从服务器map中，选取连接数最小的后端服务器进行分配请求。

优点：解决轮询算法伪均衡状态。
缺点：同一客户端，由于当时服务器的状态不同，每次请求到来可能会分配到不同的后端服务器，这就对会话一致性带来困难。

**(5)IP哈希法**
源地址哈希的思想是获取客户端访问的IP地址值，通过哈希函数计算得到一个数值，用该数值对服务器列表的大小进行取模运算，得到的结果便是要访问的服务器的序号。源地址哈希算法的代码实现大致如下：  

```
public class Hash
{
   public static String getServer()
   {
       // 重建一个Map，避免服务器的上下线导致的并发问题
       Map<String, Integer> serverMap = 
               new HashMap<String,Integer>();
       serverMap.putAll(IpMap.serverWeightMap);
 
       // 取得Ip地址List
       Set<String> keySet = serverMap.keySet();
       ArrayList<String> keyList = newArrayList<String>();
       keyList.addAll(keySet);
 
       // 在Web应用中可通过HttpServlet的getRemoteIp方法获取
       String remoteIp = "127.0.0.1";
       int hashCode = remoteIp.hashCode();
       int serverListSize = keyList.size();
       int serverPos = hashCode % serverListSize;
        returnkeyList.get(serverPos);
   }
}  

```

前两部分和轮询法、随机法一样就不说了，差别在于路由选择部分。通过客户端的ip也就是remoteIp，取得它的Hash值，对服务器列表的大小取模，结果便是选用的服务器在服务器列表中的索引值。

优点：保证了相同客户端IP地址将会被哈希到同一台后端服务器，直到后端服务器列表变更。根据此特性可以在服务消费者与服务提供者之间建立有状态的session会话。

缺点：除非集群中服务器的非常稳定，基本不会上下线，否则一旦有服务器上线、下线，那么通过源地址哈希算法路由到的服务器是服务器上线、下线前路由到的服务器的概率非常低，如果是session则取不到session，如果是缓存则可能引发”雪崩”。如果这么解释不适合明白，可以看我之前的一篇文章MemCache超详细解读，一致性Hash算法部分。

### 5.会话一致性

用户(浏览器)在和服务端交互的时候，通常会在本地保存一些信息，而整个过程叫做一个会话(Session)并用唯一的Session ID进行标识。会话的概念不仅用于购物车这种常见情况，因为HTTP协议是无状态的，所以任何需要逻辑上下文的情形都必须使用会话机制，此外HTTP客户端也会额外缓存一些数据在本地，这样就可以减少请求提高性能了。如果负载均衡可能将这个会话的请求分配到不同的后台服务端上，这肯定是不合适的，必须通过多个backend共享这些数据，效率肯定会很低下，最简单的情况是保证会话一致性——相同的会话每次请求都会被分配到同一个backend上去。

- 当客户端第一次访问服务器时，负载均衡器按照某种均衡算法将请求转发到后端服务器，后端服务器在第一次响应某客户端时，会在其头部添加Cookie,之后客户端的每次访问都会带有这个Cookie值，负载均衡器按照Cookie值分配给相应的服务器。
- 一致性哈希算法,通过一致性哈希算法实现当某一台服务器宕机或者下线时，最小影响服务器集群。

 

### 6.参考文档

>http://mp.weixin.qq.com/s/Iu50lB4lYVnr-qeVCoZWEw

>http://mp.weixin.qq.com/s/DabqB3SiTKDzQEZ2cZUnYA

>http://mp.weixin.qq.com/s/Iu50lB4lYVnr-qeVCoZWEw

 

