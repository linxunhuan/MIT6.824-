# Aurora 背景历史
Aurora是一个高性能，高可靠的数据库
Aurora本身作为云基础设施一个组成部分而存在，同时又构建在Amazon自己的基础设施之上
究竟是什么导致了Aurora的产生？
+ 最早的时候，Amazon提供的云产品是EC2
  + 它可以帮助用户在Amazon的机房里和Amazon的硬件上创建类似网站的应用
  + EC2的全称是Elastic Cloud 2
+ Amazon有装满了服务器的数据中心，并且会在每一个服务器上都运行VMM（Virtual Machine Monitor）
  + 它会向它的用户出租虚拟机，而它的用户通常会租用多个虚拟机用来运行Web服务、数据库和任何其他需要运行的服务
  + 所以，在一个服务器上，有一个VMM，还有一些EC2实例，其中每一个实例都出租给不同的云客户
  + 每个EC2实例都会运行一个标准的操作系统，比如说Linux
  + 在操作系统之上，运行的是应用程序，例如Web服务、数据库
<img src=".\picture\image95.png">


+ 因为每一个服务器都有一块本地的硬盘
  + 在最早的时候，如果租用一个EC2实例
  + 每一个EC2实例会从服务器的本地硬盘中分到一小片硬盘空间
    + 所以，最早的时候EC2用的都是本地盘，每个EC2实例会分到本地盘的一小部分
    + 但是从EC2实例的操作系统看起来就是一个硬盘，一个模拟的硬盘
<img src=".\picture\image94.png">

+ EC2对于无状态的Web服务器来说是完美的
  + 客户端通过自己的Web浏览器连接到一些运行了Web服务的EC2实例上
  + 如果突然新增了大量客户，你可以立刻向Amazon租用更多的EC2实例，并在上面启动Web服务
  + 这样你就可以很简单的对你的Web服务进行扩容
+ 另一类人们主要运行在EC2实例的服务是数据库
  + 通常来说一个网站包含了一些无状态的Web服务
  + 任何时候这些Web服务需要一些持久化存储的数据时，它们会与一个后端数据库交互
+ 所以，现在的场景是，在Amazon基础设施之外有一些客户端浏览器（C1，C2，C3）
  + 之后是一些EC2实例，上面运行了Web服务
    + 这里你可以根据网站的规模想起多少实例就起多少
  + 之后，还有一个EC2实例运行了数据库
  + Web服务所在的EC2实例会与数据库所在的EC2实例交互，完成数据库中记录的读写
<img src=".\picture\image96.png">

+ 不幸的是，对于数据库来说，EC2就不像对于Web服务那样完美了，最直接的原因就是存储
  + 对于运行了数据库的EC2实例，获取存储的最简单方法就是使用EC2实例所在服务器的本地硬盘
  + 如果服务器宕机了，那么它本地硬盘也会无法访问
    + 当Web服务所在的服务器宕机了，是完全没有问题的
      + 因为Web服务本身没有状态，你只需要在一个新的EC2实例上启动一个新的Web服务就行
    + 但是如果数据库所在的服务器宕机了，并且数据存储在服务器的本地硬盘中
      + 那么就会有大问题，因为数据丢失了
+ Amazon本身有实现了块存储的服务，叫做S3
  + 可以定期的对数据库做快照，并将快照存储在S3上，并基于快照来实现故障恢复
  + 但是这种定期的快照意味着你可能会损失两次快照之间的数据
+ 所以，为了向用户提供EC2实例所需的硬盘
  + 并且硬盘数据不会随着服务器故障而丢失，就出现了一个与Aurora相关的服务
  + 并且同时也是容错的且支持持久化存储的服务，这个服务就是EBS
+ EBS全称是Elastic Block Store
  + 从EC2实例来看，EBS就是一个硬盘，可以像一个普通的硬盘一样去格式化它
    + 就像一个类似于ext3格式的文件系统或者任何其他Linux文件系统
  + 但是在实现上，EBS底层是一对互为副本的存储服务器
    + 随着EBS的推出，你可以租用一个EBS volume
    + 一个EBS volume看起来就像是一个普通的硬盘一样，但却是由一对互为副本EBS服务器实现，每个EBS服务器本地有一个硬盘
    + 所以，现在运行了一个数据库，相应的EC2实例将一个EBS volume挂载成自己的硬盘
    + 当数据库执行写磁盘操作时，数据会通过网络送到EBS服务器
<img src=".\picture\image97.png">

+ 这两个EBS服务器会使用Chain Replication进行复制
  + 所以写请求首先会写到第一个EBS服务器，之后写到第二个EBS服务器
  + 然后从第二个EBS服务器，EC2实例可以得到回复
  + 当读数据的时候，因为这是一个Chain Replication，EC2实例会从第二个EBS服务器读取数据
<img src=".\picture\image98.png">

+ 所以现在，运行在EC2实例上的数据库有了可用性
  + 因为现在有了一个存储系统可以在服务器宕机之后，仍然能持有数据
  + 如果数据库所在的服务器挂了，你可以启动另一个EC2实例，并为其挂载同一个EBS volume，再启动数据库
  + 新的数据库可以看到所有前一个数据库留下来的数据，就像你把硬盘从一个机器拔下来，再插入到另一个机器一样
    + 所以EBS非常适合需要长期保存数据的场景，比如说数据库
+ **任何时候，只有一个EC2实例，一个虚机可以挂载一个EBS volume**
  + 所以，尽管所有人的EBS volume都存储在一个大的服务器池子里，每个EBS volume只能被一个EC2实例所使用
-------------------------------
EBS的不足
+ 如果在EBS上运行一个数据库，那么最终会有大量的数据通过网络来传递
  + 在论文中有暗示，除了网络的限制之外，还有CPU和存储空间的限制
    + 在Aurora论文中，花费了大量的精力来降低数据库产生的网络负载，同时看起来相对来说不太关心CPU和存储空间的消耗
    + 所以也可以理解成他们认为网络负载更加重要
+ 另一个问题是，EBS的容错性不是很好
  + 出于性能的考虑，Amazon总是将EBS volume的两个副本存放在同一个数据中心
    + 所以，如果一个副本故障了，那没问题，因为可以切换到另一个副本
    + 但是如果整个数据中心挂了，那就没辙了
<img src=".\picture\image99.png">

+ Amazon描述的是，EC2实例和两个EBS副本都运行在一个AZ（Availability Zone）
  + 在Amazon的术语中，一个AZ就是一个数据中心
+ Amazon通常这样管理它们的数据中心，在一个城市范围内有多个独立的数据中心
  + 大概2-3个相近的数据中心，通过冗余的高速网络连接在一起
  + 但是对于EBS来说，为了降低使用Chain Replication的代价，Amazon 将EBS的两个副本放在一个AZ中
# 故障可恢复事务（Crash Recoverable Transaction）
典型的数据库是如何设计的
Aurora使用的是与MySQL类似的机制实现，但是又以一种有趣的方式实现了加速
所以我们需要知道一个典型的数据库是如何设计实现的，这样我们才能知道Aurora是如何实现加速的
-------------------------------------
主要关注的是，如何实现一个故障可恢复事务（Crash Recoverable Transaction）
+ 主要看的是事务（Transaction）
+ 故障可恢复（Crash Recovery）
+ 首先，什么是事务？
  + 事务是指将多个操作打包成原子操作，并确保多个操作顺序执行
+ 假设我们运行一个银行系统，我们想在不同的银行账户之间转账
  + 首先需要定义想要原子打包的多个操作的开始
  + 之后是操作的内容，现在我们想要从账户Y转10块钱到账户X，那么账户X需要增加10块，账户Y需要减少10块
  + 最后表明事务结束
+ 我们希望数据库顺序执行这两个操作，并且不允许其他任何人看到执行的中间状态
  + 同时，考虑到故障，如果在执行的任何时候出现故障
    + 我们需要确保故障恢复之后，要么所有操作都已经执行完成，要么一个操作也没有执行
    + 这是我们想要从事务中获得的效果
  + 除此之外，数据库的用户期望数据库可以通知事务的状态，也就是事务是否真的完成并提交了
    + 如果一个事务提交了，用户期望事务的效果是可以持久保存的
    + 即使数据库故障重启了，数据也还能保存
+ 通常来说，事务是通过对涉及到的每一份数据加锁来实现
  + 在整个事务的过程中，都对X，Y加了锁
  + 并且只有当事务结束、提交并且持久化存储之后，锁才会被释放
  + 所以，数据库实际上在事务的过程中，是通过对数据加锁来确保其他人不能访问
<img src=".\picture\image100.png">

具体是怎么实现的呢？
+ 在硬盘上存储了数据的记录，或许是以B-Tree方式构建的索引
  + 有一些data page用来存放数据库的数据
    + 其中一个存放了X的记录
    + 另一个存放了Y的记录
  + 每一个data page通常会存储大量的记录，而X和Y的记录是page中的一些bit位
+ 在硬盘中，除了有数据之外，还有一个预写式日志（Write-Ahead Log，简称为WAL）
  + 预写式日志对于系统的容错性至关重要
<img src=".\picture\image101.png">

+ 在服务器内部，有数据库软件，通常数据库会对最近从磁盘读取的page有缓存
<img src=".\picture\image102.png">

+ 在执行一个事务内的各个操作时
  + 例如执行 X=X+10 的操作时
  + 数据库会从硬盘中读取持有X的记录，给数据加10
    + 但是在事务提交之前，数据的修改还只在本地的缓存中，并没有写入到硬盘
    + 我们现在还不想向硬盘写入数据，因为这样可能会暴露一个不完整的事务
+ 为了让数据库在故障恢复之后，还能够提供同样的数据
  + 在允许数据库软件修改硬盘中真实的data page之前，数据库软件需要先在WAL中添加Log条目来描述事务
  + 所以在提交事务之前，数据库需要先在WAL中写入完整的Log条目，来描述所有有关数据库的修改
    + 并且这些Log是写入磁盘的
+ 假设，X的初始值是500，Y的初始值是750
<img src=".\picture\image103.png">

+ 在提交并写入硬盘的data page之前，数据库通常需要写入至少3条Log记录：
  + 第一条表明，作为事务的一部分，我要修改X，它的旧数据是500，我要将它改成510
  + 第二条表明，我要修改Y，它的旧数据是750，我要将它改成740
  + 第三条记录是一个Commit日志，表明事务的结束
+ 通常来说，前两条Log记录会打上事务的ID作为标签
  + 这样在故障恢复的时候，可以根据第三条commit日志找到对应的Log记录
  + 进而知道哪些操作是已提交事务的，哪些是未完成事务的
<img src=".\picture\image104.png">

+ 如果数据库成功的将事务对应的操作和commit日志写入到磁盘中
  + 数据库可以回复给客户端说，事务已经提交了
+ 而这时，客户端也可以确认事务是永久可见的
+ 接下来有两种情况:
  + 如果数据库没有崩溃，那么在它的cache中，X，Y对应的数值分别是510和740
    + 最终数据库会将cache中的数值写入到磁盘对应的位置
    + 所以数据库写磁盘是一个lazy操作，它会对更新进行累积，每一次写磁盘可能包含了很多个更新操作
    + 这种累积更新可以提升操作的速度
  + 如果数据库在将cache中的数值写入到磁盘之前就崩溃了，这样磁盘中的page仍然是旧的数值
    + 当数据库重启时，恢复软件会扫描WAL日志，发现对应事务的Log，并发现事务的commit记录
    + 那么恢复软件会将新的数值写入到磁盘中
    + 这被称为redo，它会重新执行事务中的写操作
+ 这就是事务型数据库的工作原理的简单描述
  + 同时这也是一个极度精简的MySQL数据库工作方式的介绍，MySQL基本以这种方式实现了故障可恢复事务
  + 而Aurora就是基于这个开源软件MYSQL构建的
# 关系型数据库（Amazon RDS）
+ 在MySQL基础上，结合Amazon自己的基础设施，Amazon为其云用户开发了改进版的数据库，叫做RDS（Relational Database Service）
  + RDS是第一次尝试将数据库在多个AZ之间做复制
  + 这样就算整个数据中心挂了，还是可以从另一个AZ重新获得数据而不丢失任何写操作
+ 对于RDS来说，有且仅有一个EC2实例作为数据库
  + 这个数据库将它的data page和WAL Log存储在EBS，而不是对应服务器的本地硬盘
  + 当数据库执行了写Log或者写page操作时，这些写请求实际上通过网络发送到了EBS服务器
  + 所有这些服务器都在一个AZ中
<img src=".\picture\image105.png">

+ 每一次数据库软件执行一个写操作
  + Amazon会自动的，对数据库无感知的，将写操作拷贝发送到另一个数据中心的AZ中
+ 从论文的图2来看，可以发现这是另一个EC2实例，它的工作就是执行与主数据库相同的操作
+ 所以，AZ2的副数据库会将这些写操作拷贝AZ2对应的EBS服务器
<img src=".\picture\image106.png">

+ 在RDS的架构中，每一次写操作
  + 例如数据库追加日志或者写磁盘的page，数据除了发送给AZ1的两个EBS副本之外，还需要通过网络发送到位于AZ2的副数据库
    + 副数据库接下来会将数据再发送给AZ2的两个独立的EBS副本
  + 之后，AZ2的副数据库会将写入成功的回复返回给AZ1的主数据库
  + 主数据库看到这个回复之后，才会认为写操作完成了
+ RDS这种架构提供了更好的容错性
  + 因为现在在一个其他的AZ中，有了数据库的一份完整的实时的拷贝
    + 这个拷贝可以看到所有最新的写请求
  + 即使AZ1发生火灾都烧掉了，你可以在AZ2的一个新的实例中继续运行数据库，而不丢失任何数据
+ RDS的写操作代价极高,因为需要写大量的数据
  + 即使如之前的例子，执行类似于 x+10，y-10
  + 这样的操作，虽然看起来就是修改两个整数，每个整数或许只有8字节或者16字节，但是对于data page的读写，极有可能会比10多个字节大得多
    + 因为每一个page会有8k字节，或者16k字节
    + 或者是一些由文件系统或者磁盘块决定的相对较大的数字
  + 这意味着，哪怕是只写入这两个数字，当需要更新data page时，需要向磁盘写入多得多的数据
    + 如果使用本地的磁盘，明显会快得多
# Aurora 初探
+ 在Aurora的架构中，有两件有意思的事情：
  + 第一个是，在替代EBS的位置，有6个数据的副本，位于3个AZ，每个AZ有2个副本
    + 所以现在有了超级容错性，并且每个写请求都需要以某种方式发送给这6个副本
<img src=".\picture\image107.png">

现在有了更多的副本，为什么Aurora不是更慢了，之前Mirrored MySQL中才有4个副本。答案是:
+ **这里通过网络传递的数据只有Log条目**
  + 每一条Log条目只有几十个字节那么多，也就是存一下旧的数值，新的数值，所以Log条目非常小
  + 然而，当一个数据库要写本地磁盘时
    + 它更新的是data page，这里的数据是巨大的
  + 所以，对于每一次事务，需要通过网络发送多个8k字节的page数据
    + 而Aurora只是向更多的副本发送了少量的Log条目
      + Log条目的大小比8K字节小得多，所以在网络性能上这里就胜出了
+ 这里的后果是，这里的存储系统不再是通用（General-Purpose）存储
  + 这是一个可以理解MySQL Log条目的存储系统
+ **EBS是一个非常通用的存储系统**
  + 它模拟了磁盘，只需要支持读写数据块
  + EBS不理解除了数据块以外的其他任何事物
    + 而这里的存储系统理解使用它的数据库的Log
  + 所以这里，Aurora将通用的存储去掉了
    + 取而代之的是一个应用定制的（Application-Specific）存储系统
------------------------------------------
+ 另一件重要的事情是，Aurora并不需要6个副本都确认了写入才能继续执行操作
  + 相应的，只要Quorum形成了，也就是任意4个副本确认写入了，数据库就可以继续执行操作
  + 所以，当我们想要执行写入操作时
    + 如果有一个AZ下线了，或者AZ的网络连接太慢了，或者只是服务器响应太慢了
    + Aurora可以忽略最慢的两个服务器，或者已经挂掉的两个服务器
    + 它只需要6个服务器中的任意4个确认写入，就可以继续执行
  + 通过这种方法，Aurora可以有更多的副本，更多的AZ，但是又不用付出大的性能代价
    + 因为它永远也不用等待所有的副本
    + 只需要等待6个服务器中最快的4个服务器即可
<img src=".\picture\image108.png">

# Aurora存储服务器的容错目标（Fault-Tolerant Goals）
从之前的描述可以看出，Aurora的Quorum系统管理了6个副本的容错系统
所以值得思考的是，Aurora的容错目标是什么？
+ 首先是对于写操作，当只有一个AZ彻底挂了之后，写操作不受影响
+ 其次是对于读操作，当一个AZ和一个其他AZ的服务器挂了之后，读操作不受影响
  + 在AZ下线的这段时间，我们只能依赖其他AZ的服务器
  + 所以当一个AZ彻底下线了之后，对于读操作，Aurora还能容忍一个额外服务器的故障，并且仍然可以返回正确的数据
+ 此外，Aurora期望能够容忍暂时的慢副本
  + 如果向EBS读写数据，并不能得到稳定的性能，有时可能会有一些卡顿
    + 或许网络中一部分已经过载了，或许某些服务器在执行软件升级
    + 任何类似的原因会导致暂时的慢副本
  + 所以Aurora期望能够在出现短暂的慢副本时，仍然能够继续执行操作
+ 最后一个需求是，如果一个副本挂了，在另一个副本挂之前，是争分夺秒的
  + 因为通常来说服务器故障不是独立的
  + 事实上，一个服务器挂了，通常意味着有很大的可能另一个服务器也会挂，因为它们有相同的硬件
  + 如果其中一个有缺陷，非常有可能会在另一个服务器中也会有相同的缺陷
  + 对于Aurora的Quorum系统，有点类似于Raft
    + 只能从局部故障中恢复
    + 所以这里需要快速生成新的副本（Fast Re-replication）
  + 也就是说如果一个服务器看起来永久故障了，我们期望能够尽可能快的根据剩下的副本，生成一个新的副本
+ 这里的讨论只针对存储服务器，以及如何从故障中恢复
  + 如果数据库服务器本身挂了，该如何恢复是一个完全不同的话题
# Quorum 复制机制（Quorum Replication）
+ Aurora使用的是一种经典quorum思想的变种
  + Quorum系统背后的思想是通过复制构建容错的存储系统，并确保即使有一些副本故障了，读请求还是能看到最近的写请求的数据
  + 通常来说，Quorum系统就是简单的读写系统，支持Put/Get操作
  + 它们通常不直接支持更多更高级的操作
    + 你有一个对象，你可以读这个对象，也可以通过写请求覆盖这个对象的数值
+ 假设有N个副本
  + 为了能够执行写请求，必须要确保写操作被W个副本确认，W小于N
    + 所以你需要将写请求发送到这W个副本
  + 如果要执行读请求，那么至少需要从R个副本得到所读取的信息
+ 这里的W对应的数字称为Write Quorum，R对应的数字称为Read Quorum
+ 这里的关键点在于，W、R、N之间的关联
  + Quorum系统要求，任意你要发送写请求的W个服务器，必须与任意接收读请求的R个服务器有重叠
  + 这意味着，R加上W必须大于N（ 至少满足R + W = N + 1 ），这样任意W个服务器至少与任意R个服务器有一个重合
<img src=".\picture\image109.png">

这里还有一个关键的点
客户端读请求可能会得到R个不同的结果，客户端如何知道从R个服务器得到的R个结果中，哪一个是正确的呢？
+ 在Quorum系统中使用的是版本号（Version）
  + 所以，每一次执行写请求，需要将新的数值与一个增加的版本号绑定
  + 之后，客户端发送读请求，从Read Quorum得到了一些回复，客户端可以直接使用其中的最高版本号的数值
---------------------------------------------------
+ 为了实现上一节描述的Aurora的容错目标，也就是在一个AZ完全下线时仍然能写，在一个AZ加一个其他AZ的服务器下线时仍然能读
  + Aurora的Quorum系统中，N=6，W=4，R=3
    + W等于4意味着，当一个AZ彻底下线时，剩下2个AZ中的4个服务器仍然能完成写请求
    + R等于3意味着，当一个AZ和一个其他AZ的服务器下线时，剩下的3个服务器仍然可以完成读请求
    + 当3个服务器下线了，系统仍然支持读请求，仍然可以返回当前的状态，但是却不能支持写请求
      + 所以，当3个服务器挂了，现在的Quorum系统有足够的服务器支持读请求，并据此重建更多的副本
      + 但是在新的副本创建出来替代旧的副本之前，系统不能支持写请求
      + 同时，如我之前解释的，Quorum系统可以剔除暂时的慢副本
# Aurora读写存储服务器
+ 对于Aurora来说，它的写请求从来不会覆盖任何数据，它的写请求只会在当前Log中追加条目（Append Entries）
  + 所以，Aurora使用Quorum只是在数据库执行事务并发出新的Log记录时
  + 确保Log记录至少出现在4个存储服务器上，之后才能提交事务
+ 所以，Aurora的Write Quorum的实际意义是
  + 每个新的Log记录必须至少追加在4个存储服务器中，之后才可以认为写请求完成了
  + 当Aurora执行到事务的结束，并且在回复给客户端说事务已经提交之前，Aurora必须等待Write Quorum的确认，也就是4个存储服务器的确认，组成事务的每一条Log都成功写入了
+ 实际上，在一个故障恢复过程中，事务只能在之前所有的事务恢复了之后才能被恢复
  + 所以，实际中，在Aurora确认一个事务之前，它必须等待Write Quorum确认之前所有已提交的事务
  + 之后再确认当前的事务，最后才能回复给客户端
+ 这里的存储服务器接收Log条目，这是它们看到的写请求
  + 它们并没有从数据库服务器获得到新的data page，它们得到的只是用来描述data page更新的Log条目
  + 但是存储服务器内存最终存储的还是数据库服务器磁盘中的page
  + 在存储服务器的内存中，会有自身磁盘中page的cache
    + 例如page1（P1），page2（P2）
    + 这些page其实就是数据库服务器对应磁盘的page
<img src=".\picture\image110.png">

+ 当一个新的写请求到达时，这个写请求只是一个Log条目，Log条目中的内容需要应用到相关的page中
  + 但是我们不必立即执行这个更新，可以等到数据库服务器或者恢复软件想要查看那个page时才执行
  + 对于每一个存储服务器存储的page，如果它最近被一个Log条目修改过
    + 那么存储服务器会在内存中缓存一个旧版本的page
    + 和一系列来自于数据库服务器有关修改这个page的Log条目
  + 所以，对于一个新的Log条目，它会立即被追加到影响到的page的Log列表中
    + 这里的Log列表从上次page更新过之后开始（相当于page是snapshot，snapshot后面再有一系列记录更新的Log）
    + 如果没有其他事情发生，那么存储服务器会缓存旧的page和对应的一系列Log条目
<img src=".\picture\image111.png">

+ 如果之后数据库服务器将自身缓存的page删除了，过了一会又需要为一个新的事务读取这个page，它会发出一个读请求
  + 请求发送到存储服务器，会要求存储服务器返回当前最新的page数据
    + 在这个时候，存储服务器才会将Log条目中的新数据更新到page
    + 并将page写入到自己的磁盘中，之后再将更新了的page返回给数据库服务器
  + 同时，存储服务器在自身cache中会删除page对应的Log列表，并更新cache中的page
---------------------------------
+ 如刚刚提到的，数据库服务器有时需要读取page
  + 数据库服务器写入的是Log条目，但是读取的是page
+ 这也是与Quorum系统不一样的地方
  + Quorum系统通常读写的数据都是相同的
  + 除此之外，在一个普通的操作中，数据库服务器可以避免触发Quorum Read
  + 数据库服务器会记录每一个存储服务器接收了多少Log
    + 所以，首先，Log条目都有类似12345这样的编号
    + 当数据库服务器发送一条新的Log条目给所有的存储服务器，存储服务器接收到它们会返回说，我收到了第79号和之前所有的Log
    + 数据库服务器会记录这里的数字，或者说记录每个存储服务器收到的最高连续的Log条目号
  + 这样的话，当一个数据库服务器需要执行读操作
    + 它只会挑选拥有最新Log的存储服务器，然后只向那个服务器发送读取page的请求
    + 所以，数据库服务器执行了Quorum Write，但是却没有执行Quorum Read
    + 因为它知道哪些存储服务器有最新的数据，然后可以直接从其中一个读取数据
      + 这样的代价小得多，因为这里只读了一个副本，而不用读取Quorum数量的副本
+ 但是，数据库服务器有时也会使用Quorum Read
  + 假设数据库服务器运行在某个EC2实例，如果相应的硬件故障了，数据库服务器也会随之崩溃
  + 在Amazon的基础设施有一些监控系统可以检测到Aurora数据库服务器崩溃
  + 之后Amazon会自动的启动一个EC2实例，在这个实例上启动数据库软件，并告诉新启动的数据库：
    + 你的数据存放在那6个存储服务器中，请清除存储在这些副本中的任何未完成的事务，之后再继续工作
  + 这时，Aurora会使用Quorum的逻辑来执行读请求
    + 因为之前数据库服务器故障的时候，它极有可能处于执行某些事务的中间过程
    + 所以当它故障了，它的状态极有可能是它完成并提交了一些事务，并且相应的Log条目存放于Quorum系统
    + 同时，它还在执行某些其他事务的过程中，这些事务也有一部分Log条目存放在Quorum系统中
      + 但是因为数据库服务器在执行这些事务的过程中崩溃了，这些事务永远也不可能完成
+ 对于这些未完成的事务，我们可能会有这样一种场景
  + 第一个副本有第101个Log条目，第二个副本有第102个Log条目，第三个副本有第104个Log条目
  + 但是没有一个副本持有第103个Log条目
<img src=".\picture\image112.png">

+ 所以故障之后，新的数据库服务器需要恢复
  + 它会执行Quorum Read，找到第一个缺失的Log序号，在上面的例子中是103
    + 并说，我们现在缺失了一个Log条目
    + 我们不能执行这条Log之后的所有Log，因为我们缺失了一个Log对应的更新
+ 所以，这种场景下，数据库服务器执行了Quorum Read，从可以连接到的存储服务器中发现103是第一个缺失的Log条目
  + 这时，数据库服务器会给所有的存储服务器发送消息说：
    + 请丢弃103及之后的所有Log条目
    + 103及之后的Log条目必然不会包含已提交的事务
    + 因为我们知道只有当一个事务的所有Log条目存在于Write Quorum时，这个事务才会被commit
    + 所以对于已经commit的事务我们肯定可以看到相应的Log
    + 这里我们只会丢弃未commit事务对应的Log条目
+ 所以，某种程度上，我们将Log在102位置做了切割，102及之前的Log会保留
  + 但是这些会保留的Log中，可能也包含了未commit事务的Log，数据库服务器需要识别这些Log
    + 这是可行的，可以通过
      + Log条目中的事务ID
      + 事务的commit Log条目
      + 来判断哪些Log属于已经commit的事务，哪些属于未commit的事务
  + 数据库服务器可以发现这些未完成的事务对应Log，并发送undo操作来撤回所有未commit事务做出的变更
    + 这就是为什么Aurora在Log中同时也会记录旧的数值的原因
    + 因为只有这样，数据库服务器在故障恢复的过程中，才可以回退之前只提交了一部分，但是没commit的事务
# 数据分片（Protection Group）
+ 这一部分讨论，Aurora如何处理大型数据库
  + 目前为止，我们已经知道Aurora将自己的数据分布在6个副本上，每一个副本都是一个计算机，上面挂了1-2块磁盘
  + 但是如果只是这样的话，我们不能拥有一个数据大小大于单个机器磁盘空间的数据库
    + 因为虽然我们有6台机器，但是并没有为我们提供6倍的存储空间
    + 每个机器存储的都是相同的数据
+ 为了能支持超过10TB数据的大型数据库
  + Amazon的做法是将数据库的数据，分割存储到多组存储服务器上，每一组都是6个副本，分割出来的每一份数据是10GB
  + 所以，如果一个数据库需要20GB的数据，那么这个数据库会使用2个PG（Protection Group）
    + 其中一半的10GB数据在一个PG中，包含了6个存储服务器作为副本
    + 另一半的10GB数据存储在另一个PG中，这个PG可能包含了不同的6个存储服务器作为副本
<img src=".\picture\image113.png">

+ 可以将磁盘中的data page分割到多个独立的PG中
  + 比如说奇数号的page存在PG1，偶数号的page存在PG2
如果有多个Protection Group，该如何分割Log呢？
+ 答案是，当Aurora需要发送一个Log条目时
  + 它会查看Log所修改的数据，并找到存储了这个数据的Protection Group
  + 并把Log条目只发送给这个Protection Group对应的6个存储服务器
+ 这意味着，每个Protection Group只存储了部分data page和所有与这些data page关联的Log条目
  + 所以每个Protection Group存储了所有data page的一个子集，以及这些data page相关的Log条目
+ 如果其中一个存储服务器挂了，我们期望尽可能快的用一个新的副本替代它
  + 因为如果4个副本挂了，我们将不再拥有Read Quorum，我们也因此不能创建一个新的副本
  + 所以我们想要在一个副本挂了以后，尽可能快的生成一个新的副本
+ 表面上看，每个存储服务器存放了某个数据库的某个某个Protection Group对应的10GB数据
  + 但实际上每个存储服务器可能有1-2块几TB的磁盘，上面存储了属于数百个Aurora实例的10GB数据块
  + 所以在存储服务器上，可能总共会有10TB的数据
    + 当它故障时，它带走的不仅是一个数据库的10GB数据
    + 同时也带走了其他数百个数据库的10GB数据
  + 所以生成的新副本，不是仅仅要恢复一个数据库的10GB数据
    + 而是要恢复存储在原来服务器上的整个10TB的数据
--------------------------------------
+ Aurora实际使用的策略是，对于一个特定的存储服务器，它存储了许多Protection Group对应的10GB的数据块
  + 对于Protection Group A，它的其他副本是5个服务器
<img src=".\picture\image114.png">

+ 或许这个存储服务器还为Protection Group B保存了数据
+ 但是B的其他副本存在于与A没有交集的其他5个服务器中
<img src=".\picture\image-1.png">

+ 类似的，对于所有的Protection Group对应的数据块，都会有类似的副本
  + 这种模式下，如果一个存储服务器挂了，假设上面有100个数据块
  + 现在的替换策略是：
    + 找到100个不同的存储服务器，其中的每一个会被分配一个数据块
    + 也就是说这100个存储服务器，每一个都会加入到一个新的Protection Group中
    + 所以相当于，每一个存储服务器只需要负责恢复10GB的数据
    + 所以在创建新副本的时候，我们有了100个存储服务器（下图中下面那5个空白的）
<img src=".\picture\image115.png">

+ 对于每一个数据块，我们会从Protection Group中挑选一个副本，作为数据拷贝的源
  + 这样，对于100个数据块，相当于有了100个数据拷贝的源
  + 之后，就可以并行的通过网络将100个数据块从100个源拷贝到100个目的
<img src=".\picture\image116.png">

+ 假设有足够多的服务器，同时假设我们有足够的带宽
  + 现在我们可以以100的并发，并行的拷贝1TB的数据，这只需要10秒左右
  + 如果只在两个服务器之间拷贝，正常拷贝1TB数据需要1000秒左右
+ 这就是Aurora使用的副本恢复策略
  + 它意味着，如果一个服务器挂了，它可以并行的，快速的在数百台服务器上恢复
# 只读数据库（Read-only Database）
+ Aurora不仅有主数据库实例，同时多个数据库的副本
  + 对于Aurora的许多客户来说，相比读写查询，他们会有多得多的只读请求
  + 你可以设想一个Web服务器，如果你只是查看Web页面，那么后台的Web服务器需要读取大量的数据才能生成页面所需的内容，或许需要从数据库读取数百个条目
    + 但是在浏览Web网页时，写请求就要少的多
    + 或许一些统计数据要更新，或许需要更新历史记录
    + 所以读写请求的比例可能是100：1
    + 所以对于Aurora来说，通常会有非常大量的只读数据库查询
+ 对于写请求，可以只发送给一个数据库
  + 因为对于后端的存储服务器来说，只能支持一个写入者
  + 背后的原因是，Log需要按照数字编号
    + 如果只在一个数据库处理写请求，非常容易对Log进行编号
    + 但是如果有多个数据库以非协同的方式处理写请求，那么为Log编号将会非常非常难
+ 但是对于读请求，可以发送给多个数据库
  + Aurora的确有多个只读数据库，这些数据库可以从后端存储服务器读取数据
  + 所以，图3中描述了，除了主数据库用来处理写请求，同时也有一组只读数据库
  + 论文中宣称可以支持最多15个只读数据库
    + 如果有大量的读请求，读请求可以分担到这些只读数据库上
<img src=".\picture\image117.png">

+ 当客户端向只读数据库发送读请求，只读数据库需要弄清楚它需要哪些data page来处理这个读请求
  + 之后直接从存储服务器读取这些data page，并不需要主数据库的介入
  + 所以只读数据库向存储服务器直接发送读取page的请求
  + 之后它会缓存读取到的page，这样对于将来的一些读请求，可以直接根据缓存中的数据返回
+ 当然，只读数据库也需要更新自身的缓存
  + 所以，Aurora的主数据库也会将它的Log的拷贝发送给每一个只读数据库
  + 主数据库会向这些只读数据库发送所有的Log条目，只读数据库用这些Log来更新它们缓存的page数据，进而获得数据库中最新的事务处理结果
<img src=".\picture\image118.png">

+ 这的确意味着只读数据库会落后主数据库一点，但是对于大部分的只读请求来说，这没问题
  + 因为如果你查看一个网页，如果数据落后了20毫秒，通常来说不会是一个大问题
+ 这里其实有一些问题，其中一个问题是，我们不想要这个只读数据库看到未commit的事务
  + 所以，在主数据库发给只读数据库的Log流中
    + 主数据库需要指出，哪些事务commit了
  + 而只读数据库需要小心的不要应用未commit的事务到自己的缓存中，它们需要等到事务commit了再应用对应的Log
+ 另一个问题是，数据库背后的B-Tree结构非常复杂，可能会定期触发rebalance
  + 而rebalance是一个非常复杂的操作，对应了大量修改树中的节点的操作，这些操作需要有原子性
  + 因为当B-Tree在rebalance的过程中，中间状态的数据是不正确的，只有在rebalance结束了才可以从B-Tree读取数据
  + 但是只读数据库直接从存储服务器读取数据库的page，它可能会看到在rebalance过程中的B-Tree
    + 这时看到的数据是非法的，会导致只读数据库崩溃或者行为异常
+ 论文中讨论了微事务（Mini-Transaction）和VDL/VCL
  + 这部分实际讨论的就是，数据库服务器可以通知存储服务器说
  + 这部分复杂的Log序列只能以原子性向只读数据库展示
    + 也就是要么全展示，要么不展示
  + 这就是微事务（Mini-Transaction）和VDL
    + 所以当一个只读数据库需要向存储服务器查看一个data page时，存储服务器会小心的
    + 要么展示微事务之前的状态，要么展示微事务之后的状态，但是绝不会展示中间状态