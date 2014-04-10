# ouster开发笔记第一周

这是辞职后全心投入自己的ouster项目开发的第一周。我觉得还是记录一下每周都做了哪些事，遇到了哪些困难，什么地方浪费了过多的时间。

先总结一下，这周主要还是很外围的东西：

* 调试登陆流程
* 构思场景内行走的逻辑
* 客户端的网络处理
* 重构封包部分代码

回头一看发现真没做什么事情，遇到的问题倒不少。

* C语言的网络操作陌生
* 回到C语言手动内存管理不习惯
* 封包的代码生成没想象中顺利

登陆流程那一块，受到封包那部分代码写得还不成熟的影响，改点东西时不时就又跑不起来了，不停地debug，这是一个做的很不到位的地方。总是觉得像封包，登陆流程这类不重要，但是总是被小的细节拖往，不认真搞就跑不起来。人物行走那一块也是，到时候这个肯定是要移到服务端去的，但是不熟习客户端的实现也整合不了。看来flare的源代码也是要花时间吃透的。

客户端网络方面，不像服务端那么多条连接，而且我也不重视客户端这边，所以尽量地做简单能用吧。开始想select之类的，后面觉得连select都麻烦了，还是朝最简单地去做。目前的做法是，单独开了一条线程在后台做网络IO。对外提供ReadPacket和WritePacket接口，有读和写两个packet队列，游戏进程只要把包丢到队列中就算交付了。做网络IO的线程在后台不停地读写队列。

很简单地用setsockopt做了一个读超时，网络线程在后台做的就是一个for循环，取发送队列包进行发送，如果没有需要发送的，就进行读IO操作，读超时了就继续下一轮循环。读取一个完整的包，则对包进行解释并放到接收队列中去。这一块代码写了还没调试，估计还是花大量时间，目前的代码是多线程读写队列都没加锁的。C语言的网络操作这块还是不够熟。说到底，看得多，练得少。

封包处理算是最坑了。目前是使用msgpack的，Go那边库封装得比较好，而C这边还得自己做些事。之前图简单，先手写了登陆用到的几个包的marshal和unmarshal，但纯手写无用工作量大得惊人的，所以还是开始做代码生成。Go那边利用reflect获取类型信息，生成C这边的处理代码。解析是库帮忙做了的，但是得到的是一个msgpack_object类型，我要继续将它映射为一个C的结构体或者一个map。只能说"It's cheap, but not free"，而且每一遍出错，程序就跑不起来了，然后又得回头debug调这一块的东西，每次改很微小的一些东西，就可能整个流程重新debug。早知道这样，拿protobuff直接用或许更省事些的。

回到C语言还真是有些不习惯。主要是内存这块吧，用了太长时间Java，内存不用自己管，实在是太舒适了，Go语言有垃圾回收也还好。而C这边就不知道要有多少内存泄露了，代码写得很烂，太多地方都忘记释放的。

没写几行C代码居然就出了个越界的问题，我发现写socket返回了一个errno=9，是bad file descripor。心想怎么会呢？调试发现是fd被内存溢出覆盖了。找了一下问题，自己把自己坑了。

	struct Packet {
		uint16_t PacketId;
		union {
			struct CharactorInfoPacket;
			struct LoginPacket;
			struct SelectCharactorPacket;
			struct LoginOkPacket;
			msgpack_object_map map;
		}data;
	};

我使用的时候是类似这样子：

	struct Packet pkt;
	struct LoginPacket *login = (struct LoginPacket*)&pkt.data;
	login->XXX = xxx;	

结果发现64位机器，Packet结构体大小才24，然后才知道只有msgpack\_object\_map算大小了，其它几个结构体都没算。唉，C语言的语法盲区，编译没报错，被坑死了。