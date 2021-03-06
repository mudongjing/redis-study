**redis基本功能使用**

---------------------------

# 基础的基础

redis的操作主要使用了几种特定的数据结构，不同于mysql之类的规定了数据存储就是建立数据表，然后向其中填写数据。

使用redis没有数据表的概念，库的话勉强算有，但不能自定义名字，只是指定了库的编号，默认16个。

对于存储数据，就是在自己指定的一个数据库中，创建一个数据结构，然后直接填写数据即可。

> 所谓数据结构，就是我们常见的键值对，不同点在于其中的值，可以是普通的字符串，或是列表，哈希表，集合，有序列表

换言之，在redis上存储数据，本质可以看作是在一个哈希表中创建一个key键，然后规定了对应的值的结构，在填写数据。

比如，我们登录到redis之后，

> 通过`select`选择一个数据库，默认就是`0`号数据库，也可以是`select 5`，即选择编号为5的数据库。
>
> 选择完毕后，通过set就是创建了我们的一个类似数据表的东西，但是对于不同的数据结构，这个set是不一样的，
>
> redis中不需要先创建再使用，直接上来就是用，如果没有对应的键就会自动给你创建。
>
> > 对于最简单的结构，就是一个字符串，就使用set。`set 键 值即字符串`。
> >
> > 对于列表就是`lpush`或`rpush`，这是向列表填入数据，顺便给创建出来，而且redis所谓的列表是一个双向链表。
> >
> > 哈希表或字典，使用`hset`创建，`hset 键名 哈希表的一个键名 对应的键值`
> >
> > 集合，使用`sadd`
> >
> > 有序列表，使用`zset`,这个结构，对于值具有集合的唯一性，而每个值又被赋予一个score值，作为依据实现有序。内部实现依赖跳表结构。

除了上述最基本的几个数据结构，redis还有一个流结构，不是基本的，但是在如今是很有用处的，

> 流结构就是一种生产者/消费者模式的实体，我们把数据填入其中，另一端负责拿取做处理，也就是所谓的消息队列，Kafka的主要功能，
>
> ```
> xadd 队列名 * 键1 值1 键2 值2 ...
> # 我们填入的一条信息需要命名id，方便队列处理哪些消息是被消费过的
> # `*`就是表示消息的命名有redis自己处理
> # 消息的内容就是后面跟着的好几个键值对
> ```
>
> ```
> xdel 队列名 消息id # 删除消息需要获取对应的消息id
> xrange 队列名 - + # 获取队列中全部消息的消息 ，顺便获取对应的id
> xrange 队列名 - 消息id # 获取队列中消息id不大于指定id的所有消息内容
> xinfo streams 队列名 # 简单查看一下队列信息
> ```
>
> ```
> xread  count 消息数量 streams 队列名 0-0或$
> # 从指定的队列读取指定数量的消息
> # 0-0 指从队列头部开始读取，$表示尾部
> xread block 毫秒数 count 消息数量 streams 队列名 $
> # 表示一条阻塞命令，会在指定时间内等待队列尾部出现消息，并获取对应数量的消息
> # 如果时间是0，就代表无休止等待
> ```
>
> ```
> xgroup create  队列名  消费组名 0-0或$
> # 表示创建的消费组此后会对指定的队列从头部或尾部开始读取消息
> # 每个组关于队列有一个类似游标的东西，记录自己已经读取的消息在哪个位置，方便下次读取新的内容
> xreadgroup GROUP 消费组名 消费者名 count 消息数量 streams 队列 >
> # 这就是消费者依赖消费组读取消息，`>`就是指按照组的游标读取一个新消息给消费者，此时游标会继续移动一位
> ```
>
> 

# 设置（set）

## 锁

redis的创建的键值可以设置存活期限，比如5秒后制动删除，使用`expire 键名 时间`。

对于一些场景，可以使用redis创建锁，比如规定在redis中创建某个键即获得一个锁，销毁后其它线程再创建就获得新的锁

> 当然，内部可以使用一个随机值作为某个线程的凭证，一次控制这个键值的存活

为了满足并发以及发生异常当情况，我们需要这个键值有一个过期时间，但是极端场景下，可能创建键值与设置期限两个步骤无法合法的完成，就需要一个特殊的命令实现两个步骤的原子化操作

```
SET 键名 v值 NX EX 秒数
# NX 是值只有当前不存在对应的键名才可以操作
# EX 就是值设置过期时间，这里是秒
# 如果希望设置为毫秒数， 则使用 PX 参数替代
```

> 这里，分布式锁实际还需要考虑操作超时的问题，实际也就是前面设置凭证的工作，需要一个对应的凭证才能删除该键值。
>
> 这里为了安全，就需要使用lua脚本完成识别凭证和删除的原子化操作。

## 位图

位图实际就是比特数组，在很多场景下，我们的记录可能就是有或无的两种状态，那么一个比特就足够。位图跟重要的用处在于之后的过滤器。

在redis中，比特数组实际就是字符串，一个字符由8个比特记录，多长的字符串也就能得出由多大的比特数，实际的操作对象就是简单的字符串结构，

```
setbit 键名 数组位置 新值0或1
# 数组位置从0开始计数
```

> 需要清醒的是，比特数组就是字符串，出来把他当作比特数组逐个位地操作，必要情况也可以作为字符串整体修改

如果是修改比特数组的局部区域，比如让数组中某个区域对应的结果设置为某个值，比如0位置向后8个位置如果作为字符是a，我们可以修改这个区域的结果成为其它字符，

```
bitfield 键名 set u数量 位置 新值
# 或
bitfield 键名 set i数量 位置 新值

# u和i分别对应无符号数和有符号数
# 数量就是我们希望 从对应的位置开始向后指定数量比特的区域，将修改为对应新值
# 比如，原有的哪个区域对应的是某个字符，我们希望修改这个字符为`a`，由于`a`对应的数值为无符号数97，就有
# bitfield 键名 set u8 位置 97
# 无符号数最多负责的数量是63，有符号数则是64
```

通过`bitfield`也可以在一条命令里对一个字符串进行多个位置区域的操作

```
bitfield 键名 set i数量1 位置1 新值1 set u数量2 位置2 新值2
```

除了直接设置新值，还可以对其进行自增，

```
bitfield 键名 incrby u数量 位置 增值
# 上述是作为无符号处理，也可使有符号数
# 主要是将对应区域的结果看作一个无符号数或有符号数进行增值操作
```

> 但是自增的一个问题在于，不注意的话可能会溢出，这里有几种处理溢出的命令
>
> WRAP、SAT、FAIL。
>
> 第一个wrap是默认的行为，即折返，实际可以看作是不处理，仍然读取当前的结果，比如8位无符号数255，再加1，就溢出了，实际的结果就是当前的所有位的值都为0，那么这种行为就会读取到0。同样的，对于8位有符号数127，加1，符号位变成1，数字就成为负数，结果是-128。
>
> 第二个sat是截断行为，就是在发生溢出行为后，把当前值设置为对应情况的极限值，比如8位的无符号数250，有增加10，结果的260是溢出的，那么实际的结果就是极限的255。如果是8位的有符号数-120，有减10，结果-130也是溢出的，这里向下的极限值就是-128，也就是最后的结果。
>
> ```
> bitfield 键名 overflow sat incrby u或i数量 位置 增值
> ```
>
> 第三个fail，就是发现溢出直接不执行该命令，即结果不变化。
>
> ```
> bitfield 键名 overflow fail incrby u或i数量 位置 增值
> ```

## 事务

为了大致实现事务的功能（当然，redis做不到关系型数据库那种程度的事务），类似于mysql之类的数据库会使用`begin`说明开始事务，`commit`说明事务可以提交，`rollback`表示反悔。redis提供的命令是，`multi`开始，`exec`提交执行，`discard`反悔。

## 乐观锁

在执行事务的时候，不希望其中的变量在实际操作时已经被改变，就需要一个方法监视，使用`watch`命令，

```
watch 变量名 # 就这样，一旦发现这个变量已经被修改，那么对于当前的事务操作而言就会执行失败，返回null
# 这个命令需要放在`multi`命令前面
```

## 异步处理

redis是一个极其注重效率工具，自己不允许任何操作占据太多时间，比如删除一个键，如果这个键本身就是一个较大的元素，彻底的删除是稍微费点时间的，可以使用unlink来删除，

```
unlink 键名 # 这个命令会在后台一个独立的线程中做删除操作，不会影响到主线程
```

类似地，如果要清空整个redis的数据库，也可异步执行

```
flushall async 
# 只清空某个数据库
flushdb 库编号
```

包括redis自己的AOF日志的生成也是一个异步线程负责操作，避免干预主线程



# 提取

## 地理位置比较

如果我们知道了一个地理位置的经纬度，另外还有一堆其它的经纬度数据，我们想要提取其中最接近这个位置的那一个，普通的计算方式耗时耗力，就需要使用一种叫`GeoHash`的算法，将二维的信息降维到一维的数据上，复杂度直接下降，而redis就直接提供了这种命令。

我们可以添加元素名称以及对应的经度纬度，也可以得出两个元素的地理距离，也可以计算某一区域内存在哪些元素。

> 使用`geoadd`可以添加元素
>
> ```
> geoadd 集合名称 经度 维度 元素名称
> # 也可以一次向集合添加多个元素
> geoadd 集合名称 经度1 维度1 元素名称1 经度2 维度2 元素名称2 ...
> ```
>
> 计算距离
>
> ```
> geodist 集合名称 元素1 元素2 距离单位
> # 单位可以米m，公里km，英里ml，英尺ft
> ```
>
> 提取某一元素的坐标
>
> ```
> geopos 集合名 元素名
> # 也可以一次提取多个元素地址
> geopos 集合名 元素名1 元素名2 ...
> ```
>
> 搜索指定区域，主要是指定一个元素划定一个半径区域，找出指定数量的元素，并按距离正序或逆序输出
>
> ```
> georadiusbymember 集合名 元素名 距离数 单位 withcoord withdist count 指定的输出的元素数量 asc或desc
> # withcoord 是给出输出元素的位置坐标
> # withdist 是给出输出元素与圆心元素的距离
> # 输出的元素会包含圆心元素
> ```

## 提取键名

如果我们建立的键太多，需要查看存在哪些相似的键名，可以使用`scan`进行搜索

```
scan 游标 match 匹配模式 count 当前实现最大数量
# 游标一般在最开始就是0，匹配模式就是类似正则表达式，利用游标可以向分页一样一页页地看结果，可以规定一页的最大记录数
# 比如
scan 0 match abc* count 10
# 就是搜索前缀是`abc`的结果，一页最多10条记录
# 结果会在最开始给出一个数字，就是下次的游标，如果最后返回的游标数字是0，就是看完所有结果了
```

> 所谓的游标实际是因为，前面不断提及redis本身就是一个键值对，就是一个哈希表，类似java中的hashMap，它也是一个数组加链表的结构，游标实际就是对应的数组的索引。
>
> 也就是我们在redis负责存储键名的哈希表上搜索对应的结果，而哈希表一个横着的数组，每个索引下面垂着一条链表，我们从数组0位置开始查询，直到当前的结果数量达到最大数量，停止，并返回需要继续搜索的数组索引，就是我们得到的游标。

此外，储类淡出的`scan`，针对其它数据结构，还有`zscan`、`hscan`、`sscan`

## 测试

如果项目上线，我们可能需要提前判断redis在当前环境下的工作效率，比如一些操作大概的执行效率，也就是`QPS`指标，

```
redis-benchmark -t set -q # 这里就能获取`set`指令每秒能执行的次数
```



# 统计

## HyperLogLog

这是一个去重统计工具，当我们向一个集合中添加数据，可以获得其中的元素个数，但是如果元素数量很大，集合的空间则极为庞大，如果我们只是需要一个大致的数量的话，这个代价是不值得的，而这里的结构就是用极小的空间代价给出较为精确的结果。

> 之所以是大致，自然是算法具有统计性质，而导致存在误差，但是对于大数据而言，且不要求极高的进度的情况下，是可以接受的。

```
pfadd 键名 元素 # 添加元素
pfcount 键名 # 获取元素个数
# 如果一开始有多个这样的键在添加元素， 当出现需求希望几个键的值合并
pfmerge 键1 键2 ...
```

### 内部实现原理

首先，最基础的结构还是使用了位图充当数据存储的容器。

其次，如何估算大致的元素数量，

> 这里先是一个最粗略的估计，计算机中的大量独立的信息，我们如何通过一个统计的方式合理的猜测它们的数量，自然需要从这些信息中提取一些公有的信息作为一个概率的统计依据，而计算机中的信息都是比特信息组成的，大不了就用这些0和1作为猜测的依据。
>
> 由于这些元素是独立的，那么它产生的0和1也是没有相互关系的，于是HyperLogLog发明者，就用一个信息对应的比特数组中低位的0的连续个数作为随机的对象。
>
> > 其思想大致就是，如果我们抛硬币，我们只要反面，如果出现正面，就算一个回合结束，那么，我们抛出一个反面的概率很大，就是0.5，也就是两个回合就有极大的可能得到一个这样的结果，反过来，如果对方说自己在这个游戏中，最好的结果是只抛出了连续一次反面，那我们则有理由怀疑他是不是就玩了两三个回合。
> >
> > 那么延伸一下，如果对方告诉我们他的最好记录是连续抛出10此反面，我们很难相信他只玩一次就有这样的成绩，这种概率是$\frac{1}{2^{10}}$,怎么也得玩个1000多场才合理一点。
>
> 这就是发明者最开始的估算策略，取所有元素比特数组中低位0的最大连续数`k`，那么$2^k$就是当前比较合理的估算值。

最后，我们需要意识到，可能就是存在一个好运气的玩家，因此，需要对前面估算方法做调整。

> 调整的方式为，不再是简单地从所有元素中找哪个连续0最多的数，而是分配出大量的位置，比如一万个或几百万个，元素还是作为一个比特数组，规定了前面几个位的比特数作为该元素归属的位置，如此就把大量的元素分配到我们固定数量的不同位置中。
>
> 可能存在多个元素分配在一个位置的情况，这无所谓，最终需要提取每个位置中连续0最大的数，记为$m_{j}$,j是位置编号，令M是位置数量，有
> $$
> E=\alpha_{M}*M^2*(\sum_{j=1}^{M}2^{-m_j})^{-1}
> $$
> $E$就是当前的一个大致估值，$\alpha_M$是一个关于位置数量的常数，此后还需要对E关于其大小关系在做调整，但这里没必要再详述了。
>
> 最关键的就是，通过分配位置的方式以及求调和平均数的方式削弱元素中极端值的影响。

在redis中使用位图来存放每个位置对应的最大连续0的个数，用6个bit记录，总共16384即$2^{14}$，于是就需要12K的空间大小，相对而言，这个代价键简直可以忽略不记。

> 但redis还不满足，对于16384个位置而言，几百或几万个元素数量根本不够看，大量的位置都对应的0，可以通过记录成片的0来减少额外的存储空间，等到元素数量达到阈值，会自动成为12K的样子。

## 布隆过滤器

前面的HyperLogLog是单纯地计算元素有多少，而过滤器则是需要指导给定的元素是否之前加入过，同样的，如果简单的使用集合结构，大数据情况根本无力支持。

布隆过滤器同样是以相对小的存储代价实现基本的功能，显然，最终的功能是略有误差，但实际影响不大。

> 使用场景是，如果我们逛视频网站，可能点开一个视频后，页面会显示这个视频我们曾经看过了，回想一下也许是几年前看了，那么几年下来，看的频繁一些，看过的视频数量怎么也得有个一两万个，公司显然不会为每个用户都记录这些看过的视频的连接或其它信息，无论如何，只要记录了视频的特有信息，总会随之时间而不断扩大存储空间，是不值得的。

布隆过滤器，就是以一个固定的存储空间，应对大量的这类信息。而误差在于，它判断这个视频看过，实际可能没看过，但判断没看过的，就一定没看过。

如果需要使用，可以使用docker直接安装`redislabs/rebloom`，这是包含了布隆的redis。

> 或者下载这个模块，`git clone https://github.com/RedisLabsModules/redisbloom.git`，编译一下，
>
> 启动redis的时候加载它，比如`redis-server --loadmodule ./redisbloom/rebloom.so`.
>
> 或者直接在`redis.conf`文件中写入`loadmodule redisbloom.so的位置`，以后启动就自动加载对应的模块。

基本的使用就是`bf.add`、`bf.exists`.

java操作可以添加依赖[JRedisBloom](https://github.com/RedisBloom/JRedisBloom)

```
 <dependency>
     <groupId>com.redislabs</groupId>
     <artifactId>jrebloom</artifactId>
     <version>2.1.0</version>
 </dependency>
```

```
import io.rebloom.client.Client

Client client = new Client("localhost", 6379);
client.add("simpleBloom", "Mark");
// Does "Mark" now exist?
client.exists("simpleBloom", "Mark"); // true
client.exists("simpleBloom", "Farnsworth"); // False
```

### 布隆过滤器原理

简单而言，布隆过滤器就是为加入的元素计算多个不同的哈希值，对哈希值取对应的比特数组添加到位图中，如果给定一个元素，计算其多个哈希值，每个哈希值在映射出对应的一个整数，作为位置索引，将位图对应的位置置为1，如果一个元素通过计算发现它对应的几个位置索引，在位图上均为1，则大概率这个元素是以前加入过的，当位图加入的元素太多或巧合时，则可能发生误判。

为了减小误判的几率，需要看情况设计自己的过滤器。首先自己估计一下自己的项目可能会包含多少元素`n`，然后给一个自己能接受的误判率`f`。

我们需要的结果是，我们需要准备多少比特数`l`，以及不同的哈希函数要准备多少`k`。有下列的简单计算公式
$$
l=-\frac{n\cdot lnf}{(ln2)^2}\\
k=ln2\cdot \frac{l}{n}
$$
