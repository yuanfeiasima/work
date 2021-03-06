整理了一下519抢购的技术点，只是一些皮毛，分享给大家，希望是抛砖引玉，大家一起共同进步。  





大前提：519抢购的要求是10秒以内完成20万订单的抢购。



为了这个目标，进行了如下两个方面的优化：

一 程序tps（每秒处理请求数），程序抗并发能力的优化。

二 集群tps优化，集群抗并发能力优化。



先说单程序tps优化：

影响单个程序tps的关键点如下：





1 json的序列化与反序列化，非常消耗cpu，尤其是没有用fastjson的时候，cpu彪的非常高。

解决方案：减少序列化和反序列化操作，能缓存的尽量走缓存，尽量把要频繁使用的序列化或者反序列化好的结果数据存到缓存中，减少序列化和反序列化的执行次数。



2缓存的使用技巧：

  不涉及到全局性的缓存数据，都尽量使用本地缓存（谷歌的开发包提供了很优秀的本地缓存功能）

  设计到全局性的缓存，比如统一的库存，统一的售罄状态等，可以使用redis等缓存方案。

  如果数据源在数据库中，并且要求实时或者近似实时的查询，其实也可以通过评估延迟时间来使用缓存。

  比如某些查询操作需要实时去数据库中查询，如果有100台web服务器会查数据库，每台的tps是5000的话，总tps也有50万。

  数据库是绝对扛不住的，即使做了读写分离。 而且这种实时查询，再高并发下主从同步很可能都是分钟级别的延迟。 对数据库的持续压力是非常大的。

  此时就可以做一个评估了，比如我们允许接收数据有1秒钟的延迟，那么我们就可以请求来的时候，先从数据库中查出数据，然后存到本地缓存中，本地缓存是非常快的，使用时的时间消耗基本可以忽略不计。 那么 100台机器 每秒只需要请求数据库100次就可以了，比50万节省了太多了。



  另外如果想提升redis的性能，第一要有一个足够高频的单核cpu（因为redis主要是用单线程模式），外加足够的内存。 

  可以给redis部属自带的哨兵，让redis有一定的容灾能力。



3数据库连接池的优化

  在高并发场景下，数据库是我们永远的痛， 数据库有几个重要的稀缺资源，数据库连接数（一般的mysql的数据库实例，最好是2000左右的连接数，如果连接数再提高，会影响数据库的性能，但也不是绝对的，有些时候可以适当的牺牲连接数换取服务器端更高的并发处理能力来提升tps，这里就需要通过压测找到那个性能最好的平衡点了）。  另外对于数据库连接池的配置，一般我们会使用阿帕奇的开源工具dbcp，它可以设置三个数量 初始连接数，最大连接数，最小连接数。 建议把这3个设置成一样的，这样会减小高并发时连接池新增或销毁链接带来的开销。  dbcp还有很多优化的技巧，这里不再说了，大家可以百度一下。



4 我理解的一个完整的单机性能模型：   

  当一大批请求访问到web容器（tomcat或者resion）时， web容器会有两个队列，第一个队列是能够得到资源执行处理的队列（可以通过调整web容器的线程数来调整队列的大小，其实线程数越多，cpu就会被越多的线程使用，所有的请求都会变慢。一般来说基于某个程序，再服务满足执行时间时，对应的线程数最佳。  理论值是 线程数是 cpu核数的 个位数倍数 比较好 ），第二个队列是排队等待的队列。  当请求数比第一个队列承受的并发高的时候，请求会进入到排队中，当排队的队列也满了之后，请求会直接返回50x错误。



  web容器还可以调整能够处理的最大线程数，再压测的时候我们做过调整，1024，2048，8192.  再我们用恒定高并发压的时候，如果cpu是瓶颈，那么调这个属性几乎对tps没太多帮助。 【web容器内部的模型纯属猜测，大家有兴趣可以细细研究】



  数据库连接数设置多少最合适呢，其实就是能够与服务所需要的连接数匹配。 什么是匹配呢，就是没有线程因为拿不到数据库连接而进行等待。



5 性能分析的工具：jprofiler

    它可以分析cpu的高消耗位置，线程等待的位置，内存泄露的位置等。  我们主要优化的其实就是这些点， 不要让线程消耗太多cpu，不要让线程因为稀缺资源而等待（用缓存替代数据库就是干这个的），不要有内存溢出。

    （友情提示，这个软件非常消耗cpu，大概自己就能吃60%的cpu，开启的时候请让并发数小一点，50左右就够了）



6 网络资源：

    当我们单机的tps达到5000+的时候，其实我们消耗的网络带宽是非常大的，那么此时启动服务器端的Gzip就非常有用了，它会多消耗一些cpu资源，但是会节省很多网络带宽。   我们最终可以根据我们手头的硬件资源来评估，到底是要cpu还是要带宽。



7 分库分表：

    分表要解决的问题是mysql单表存储数据最大量的问题，当单表数据量过大（超过1000万）时，对表操作时性能会开始下降。

    分库要解决的时硬件性能的问题。

    分表的方法更多的需要借助中间键，或者代码中有一些逻辑。

    分库则可以通过部属的方式简单实现。 比如同一套代码， 部属的时候可以 部属十组，每组服务器上绑定不同的host去连不同的数据库。 然后再web服务器之上用nginx来基于某个值做哈希让请求总能打到唯一的一组服务器上。 这种方式运维的压力会大，但程序之需要一套代码。



8 硬件上需要监控的参数：包量（主要用于计算网卡是否跑满），连接数（结合服务器端口数来计算是否占满），各个单体之间的连接是长连接还是短连接。







影响集群tps



1 在集群环境下，由于resion的海量增加，很多其他节点都可能成为瓶颈， 比如lvs，nginx，网卡，redis，数据库等。

lvs 和 nginx 出现瓶颈后，也是从cpu，网卡，连接数，包量，流量等方向查起。  

redis出问题主要看内存是否达到瓶颈， cpu是否达到瓶颈。    redis是很需要一个强劲的cpu的，我们使用的都是单线程模式的redis，比较稳定。

数据库瓶颈：一般数据库瓶颈有几个， 连接数资源（一般2000左右最好，超过了就开始有性能损失，达到4000就对数据库操作有非常大的影响了）





2 分流：把流量转移，比如把静态内容发布到cdn中，来分担服务器端的压力。  把同一个resion中的服务拆分到其他resion中，分开部署，使用独立的数据库资源等。



3 限流：直接对流量进行限制，比如再nginx中可以设置基于ip或者location的限流，访问频次控制等，比如一分钟，某个url下的某个参数（一般是user\_id）能访问服务几次。



4 降级：直接把服务中不重要的功能去掉，或者直接去掉不重要的服务。系统处于可运行状态，但是边缘功能暂时不展示或无法使用。 以保证主交易流程的畅通无阻。



5容灾：对于核心系统来说，容灾是一个很重要的话题。 首先部署层面同一个服务不能全部部署在同一个宿主机的虚机上，同业务的宿主机不能再机房再同一个机柜中，同一套服务不能只在一个机房中。 机房间要有快速切换的方案，服务切换时要有有数据同步的机制，保证数据的有效容灾。



----------------运维视角看性能（这个是我向集团的李爽同学请教的，再这里感谢一下李爽同学）-------------



一 cpu篇

  通过监控能够看到cpu的各项指标，比较重要的有 cpu idel\(cpu空闲\)，  100%-cpuidle=cpu使用量。

  cpu io wait： cpu等待磁盘io所消耗的百分比，这个值如果高就说明磁盘可能有问题了。 这个可以和磁盘的swap交换量结合起来一起看。（ 命令：free -m 看swap交换）

  cpu user：程序使用的cpu





二 监控的load性能图标。load这个值是用cpu，磁盘，内存等一起计算出来的一个值。     一般load数值\*cpu核数\*3&gt;12 就说明性能已经不行了。



三 内存

  内存主要就是通过swap置换量来分析，如果成百上千的swap置换，就说明内存已经不足，开始大量使用磁盘代替内存，此时往往 cpu io wait 也会高很多。

  （free -m）用来看swap置换量等。

四 网络

  time wait 等待的连接

  close 关闭的连接

  Estable 目前已经建立的链接

  y..     半连接



  如果time wait 比较多，可以考虑把短连接改成长连接，当然也是基于具体场景的。

  （命令：ss -s 看连接数，  ss -an  看其他的）



五 网卡

  消耗网卡的主要是两个东西，1 流量 2 包量

  流量消耗的是网卡的吞吐亮，比如千兆网卡， 流量上限就是1G。

  包量消耗的时网卡的性能，一般一台虚机 10万左右的包量基本就是极限了，再多久可能有丢包。  可以用  w -get命令 来实验是否丢包。



  可以通过给网卡做bading， 把多个网卡绑在一起来提高网卡性能。  理论可以绑定无限多个，目前我们是2个或者4个绑定在一起。



六 磁盘

  一般磁盘的性能瓶颈是看磁盘的iops （一秒多少次磁盘读写）  磁盘的io wait比较高的时候，就是磁盘有瓶颈的时候。   

  一般此时，cpu io wait 也会高。



七 端口数

  我们的七层代理（nginx） 一个ip可以有65535个端口。  这个量很可能不够。 解决的办法是nginx做多ip回源，比如nginx用多个ip吧请求upstream到下面的resion，这样就解决了端口数不够的问题。 也可以增加nginx的数量。

  （命令  ss -s 可以看有多少端口数）



八 主机虚拟化

  一个性能好的主机可以虚拟出很多的虚拟机，一般虚拟化之后，总体的性能损耗是20%-30%左右，但是带来的好处是可以做隔离，带来更多端口数，减少硬件资源浪费。

  虚拟化之后其实网卡也被虚拟化了，所以虚拟化之后的主机对于包量的处理能力会低，一般10万左右的包量就基本到上限了。



