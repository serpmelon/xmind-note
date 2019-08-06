
## 基本概念
1、cluster：es集群通过唯一名称区分各个集群，每个节点需要通过集群名称加入集群；不同环境的集群名称尽量不要一样，可以加入环境标示，比如xxx-dev，xxx-prod；
2、node：节点用来存储数据和索引；节点在创建时默认名称是uuid，可修改，与集群一样，名称也是节点区分的标示；节点会默认加入名称为“elasticsearch”的集群；
3、index：index是具有相似特点的document的集合，比如可以创建顾客index，订单index；index名称必须唯一并且小写；
4、type（deprecated）：类似index，是在index下再分一层；已经废弃，在7版本已经没有了；
5、document：document是可以被索引到的最基础的存储单元，比如为一个客户创建一个document，为一个订单创建一个document；document中的数据是通过json展示的；
6、shard：es可以存储大量的数据，这主要依靠了shard的能力，将一个index划分为多个shard到多个node，这样有助于分担单个node的存储压力，也对查询效率有提高；
7、replica：一说到cluster就必须提到高可用性，为了防止node挂掉导致系统崩溃，我们可以将index备份多个replica；一旦备份，就至少存在两个shard，一个是原始的一个是备份的；

## 配置

es只需要很少的配置，具体包括
1、elasticsearch.yml 用来配置es
2、jvm.options 用来配置jvm参数
3、log4j2.properties 用来配置日志系统

### jvm配置
我们基本不需要配置jvm参数，如果需要可能就是配置下堆的大小。jvm.options中的格式如下：
1、空白行会被忽略；
2、# 开头的是注释行，会被忽略；
3、以“-”开头的会被视为jvm参数；
4、以数字和“：”开头的会被认为是针对某一个jdk版本的参数，eg：8:-Xmx2g，表示只有jdk版本是8的时候才有效；如果设置jdk8或以上有效则格式为 8-:-Xmx2g；8后面的“-”其实表示范围，
所以如果要表达jdk8和jdk9范围内使用，则格式为 8-9:-Xmx2g
5、除了上面的所有格式都是不允许的

// 中间部分后期补充，先了解dsl

## DSL

### Query and Filter context
#### query context
query context 用来模糊查询，查询结果和你的参数匹配度高的结果，这个匹配度通过一个分数_score计算；

bool	没有must的话最少也要有一个should
	must	必须包含
	must_not	必须不能包含
	should	有的话会加分

match
	属性如下
	operator 
		and
		or
	analyzer 分词器
	lenient 忽律异常（宽容的）
		true
		false
	

#### filter context
filter 相反，就是准确匹配，查询结果是或者不是，没有近似这种情况；

term
	用于精确指定匹配哪些值；
terms
	可以针对一个字段匹配多个值

bool
	must - and
	must_not - not
	should - or
range


## 数据
### 元数据
_index
_type
_id
_source
	数据返回的地方，可以通过设置_source查询某个字段
_version
	document版本，更新数据时，是创建一个新的数据，并将版本更新，原数据不会被马上删除，但是也不能被访问，在未来某一时刻，
	es会删除原数据（可能是存储不够的时候）。
	为了在创建数据的时候不会覆盖已经存在的数据，我们可以不指定_id，让es自动生成；或者在请求中设置为创建模式：op_type=create,
	或者/_create;成功会返回201，错误会返回409。

## 搜索

### 空搜索
/_search
	返回所有数据
	hits
		表示击中的数据，及满足查询结果的数据
		total
			表示总共有查询到多少数据
		hits数组
			具体的数据，包含每条数据的各种原数据
			_score
				es是根据匹配程度打分排序的，这里记录的是该数据的评分
		max_score
			数据中的最高分
	took
		查询花费的毫秒数
	_shards
		分片相关的信息，分片数量，成功节点数量等
	time_out
		是否超时，我们可以在请求的时候限制超时时间，但是超时并不会停止查询，只是将信息返回，es还是会继续查询。所以不可以通过设置
		超时时间停止es查询

## 数据备份模型 data replication model
primary-backup model
	一共两种分片，一种是primary shard，只有一个；其他的分片 replica shard；primary shard作为index操作的主入口，一旦接收index操作，
	primary shard会将结果给replica shard，让他们更新；
如何保证primary shard高可用

### basic write model
译文：
	每一次索引操作都会首先路由到对应的副本组，通常情况下是根据document id。当副本组指定好后，会把索引操作给该组的primary shard，primary
	负责校验索引操作正确性和发送索引操作给其他副本。因为副本可能存在离线的情况，所以es维护了一份副本列表，该列表上的所有副本都需要接收信息。
	该列表被称为 in-sync copies，由master node维护。in-sync copies保证执行所有用户发送的索引和删除操作。primary负责维护这样的一种恒定（invariant）规则，
	因此需要发送每次操作给列表中的副本。
	primary操作流程如下：
	1、检测操作正确定，如果结构有问题则拒绝。
	2、本地执行操纵相关document，某些情况下也会拒绝执行，比如某个字段长度过长。
	3、发送该操作给当前in-sync copies中的副本，如果有多个副本则并行发送。
	4、副本正确执行操作，并通知primary，primary才会告知客户端执行成功。

#### failure handling
译文：
	在操作过程中错误是无法避免的，比如磁盘损坏、节点掉线。虽然可能不会很频繁，但是primary需要对这些处理。

	当primary出现问题时，它会发送信息告知master node。索引操作会等待master node在副本中选择一个新的primary（默认一分钟），之后
	会将该操作发送给新的primary。master也会监视所有node的健康状态，可能会主动将primary降级。这种情况通常发生在primary由于网路原因
	被孤立。

	一旦上述操作成功后，primary就需要处理副本在操作过程中可能会出现的问题，问题可能出现在副本也可能是网络问题。所有这些情况的统一解约方案：
	in-sync copies中的副本如果丢失了某个操作，这个副本会被标记。为了避免破坏一致性，primary会发送信息给master请求从in-sync copies中删除该副本，
	只有master执行了完删除操作，primary的操作才算完成。master会挑选其他的node作为新的副本。

	在发送操作给副本的过程中，primary也会校验自己是不是还是primary。如果一个primary已经被孤立了，那么在它意识到这个之前还会继续处理操作，继续
	发送操作给副本，副本在接收到旧的primary发送的操作后会响应拒绝信息，当该primary接收到拒绝信息后就知道自己已不再是primary，但是它还不死心，
	它会询问master情况，然后知道自己已经被取代，最后会将操作路由给新的primary～悲哀。。。

### basic read model
译文：
	在es中可以通过ID实现轻量的读取，也可以通过复杂的聚合操作实现比较重的读取（消耗一定的cpu）。primary backup model一个很重要的优点是保证了所有副本
	是一样的，所以，in-sync copies中的每一个副本都足够支持读取操作。

	当一个node接收到一个请求后，该node就需要负责发送该请求到拥有对应shard的node，核对结果，并将最终响应返回给客户端。我们把这个node叫做这个请求的coordinating node。
	大概的处理流程如下：
	1、处理相关shard的请求。大部分情况下，查询操作都会涉及多个index，这些index可能会存放在多个shard中，所以通常情况下，node需要从多个shard中读取数据。
	2、在相关shard中选择一个副本，可以是primary也可以不是普通副本，默认情况下，es会轮询选择。
	3、发送shard level请求给副本
	4、整合结果，在通过ID读取时，会指定shard，所以不需要这一步

#### shard failures
译文：
	当shard响应请求失败，coordinating node会继续发送请求给该副本组的其他副本。这样的错误在没有有效的副本的情况下会重复发生。
	为了保证快速响应，下面这些API会发送有部分结果的响应当一个或多个shard失败。
	search
	multi search
	bulk
	multi get
	响应的http状态码还是200，错误信息会在timeout或者_shard字段展示。
#### a few simple implication
译文：
	下面这些步骤展示了es作为一个系统的读写操作；因为读写是可以并行执行的，所以下面的步骤对读写都有影响；
	1、高效读
		在正常情况下，一个读请求只会在对应的复制组执行一次读操作。只有在错误的情况下才会在多个shard副本中执行多次查询。
	2、read unacknowledged 读未提交（感觉acknowledge相当于提交的意思）
		因为primary在处理写操作的时候会先在本地处理，然后将操作发送给副本，所以在执行读操作的时候，有可能选中primary进行读，所以
		会出现读到unacknowledged的情况。
	3、默认两份副本
#### failures
译文：
	一个shard可以拖慢整个索引操作
		因为primary在执行索引操作时需要等待所有副本操作都完成才算完成索引，所以一个shard可能会拖慢整个索引的速。这是我们为了保证
		高效读取付出的代价，当然如果查询的时候运气不好，刚好查询到这个shard也会导致速度慢。

	脏读
		一个孤立的primary可能会执行没有承认的写操作。因为在primary在意识到自己被孤立之前是可以操作的。在某个时刻，孤立的primary完成写
		操作，同时有读取操作在该node上执行，这样就会造成脏读。es为了减少这种情况，primary需要与master维持一个心跳（默认每秒），而且拒绝
		执行没有master同意的索引操作（那这样还没有杜绝么？）
## index API
index操作会将document以json形式存储在特定的一个index下，这个index可以指定也可以由es自动生成；
document可以指定version，默认情况下，version从1开始，每次更新或者删除都会加1。此时version的type是internal，我们可以指定type为external，
然后使用我们自己维护的version，比如存储在数据库中。我们还可以指定shard（routing）或者设置如果存在则不插入（op_type=create）等。

### wait for active shards 
这里没看懂，先翻译下
译文：
	为了提供系统的写操作的弹性，index操作在执行之前可以指定必须存在几个有效的shard。如果需要的shard数量没有达到，那么系统会一直尝试，
	直到数量达到或者超时。默认情况下，写操作只需要primary有效，也就是wait_for_active_shards = 1
