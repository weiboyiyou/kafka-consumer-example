Kafka 0.9版本开始推出了Java版本的consumer，优化了coordinator的设计以及摆脱了对zookeeper的依赖。社区最近也在探讨正式用这套consumer API替换Scala版本的consumer的计划。鉴于目前这方面的资料并不是很多，本文将尝试给出一个利用KafkaConsumer编写的多线程消费者实例，希望对大家有所帮助。
    这套API最重要的入口就是KafkaConsumer(o.a.k.clients.consumer.KafkaConsumer)，普通的单线程使用方法官网API已有介绍，这里不再赘述了。因此，我们直奔主题——讨论一下如何创建多线程的方式来使用KafkaConsumer。KafkaConsumer和KafkaProducer不同，后者是线程安全的，因此我们鼓励用户在多个线程中共享一个KafkaProducer实例，这样通常都要比每个线程维护一个KafkaProducer实例效率要高。但对于KafkaConsumer而言，它不是线程安全的，所以实现多线程时通常由两种实现方法：
    
1 每个线程维护一个KafkaConsumer
2  维护一个或多个KafkaConsumer，同时维护多个事件处理线程(worker thread)


当然，这种方法还可以有多个变种：比如每个worker线程有自己的处理队列。consumer根据某种规则或逻辑将消息放入不同的队列。不过总体思想还是相同的，故这里不做过多展开讨论了。
　　下表总结了两种方法的优缺点： 
 	优点	缺点
方法1(每个线程维护一个KafkaConsumer)	方便实现
速度较快，因为不需要任何线程间交互
易于维护分区内的消息顺序	更多的TCP连接开销(每个线程都要维护若干个TCP连接)
consumer数受限于topic分区数，扩展性差
频繁请求导致吞吐量下降
线程自己处理消费到的消息可能会导致超时，从而造成rebalance
方法2 (单个(或多个)consumer，多个worker线程)	可独立扩展consumer数和worker数，伸缩性好	
实现麻烦
通常难于维护分区内的消息顺序
处理链路变长，导致难以保证提交位移的语义正确性 
 
下面我们分别实现这两种方法。需要指出的是，下面的代码都是最基本的实现，并没有考虑很多编程细节，比如如何处理错误等。


参考：http://www.cnblogs.com/huxi2b/p/6124937.html
