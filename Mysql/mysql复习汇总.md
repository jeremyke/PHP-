 ## 1锁

 数据库锁定机制简单来说，就是数据库为了保证数据的一致性，而使各种共享资源在被并发访问变得有序所设计的一种规则

 #### 1.1 锁分类

 ###### 1.1.1 按照粒度分

 - 表锁

 锁定整张表，是MySQL各存储引擎中最大颗粒度的锁定机制。由于表级锁一次会将整个表锁定，所以可以很好的避免困扰我们的死锁问题，但是并发度小。使
 用表级锁定的主要是MyISAM，MEMORY等一些非事务性存储引擎。

 - 行锁

 锁定操作行，是MySQL各存储引擎中最小颗粒度的锁定机制。由于锁定颗粒度很小，所以发生锁定资源争用的概率也最小，并发度高。但是由于锁定资源的颗
 粒度很小，所以每次获取锁和释放锁需要做的事情也更多，带来的消耗自然也就更大了。此外，行级锁定也最容易发生死锁。使用行级锁定的主要是InnoDB存储引擎。

**总结：**

```text
表级锁：开销小，加锁快；不会出现死锁；锁定粒度大，发生锁冲突的概率最高，并发度最低；

行级锁：开销大，加锁慢；会出现死锁；锁定粒度最小，发生锁冲突的概率最低，并发度也最高；   
```

 #### 1.2 锁详解

 ###### 1.2.1 表锁的详解

1.MySQL表级锁的锁模式

MySQL的表级锁有两种模式：表共享读锁和表独占写锁。锁模式的兼容性：
```text
对MyISAM表的读操作，不会阻塞其他用户对同一表的读请求，但会阻塞对同一表的写请求；

对MyISAM表的写操作，则会阻塞其他用户对同一表的读和写操作；

MyISAM表的读操作与写操作之间，以及写操作之间是串行的。当一个线程获得对一个表的写锁后，只有持有锁的线程可以对表进行更新操作。其他线程的读、
写操作都会等待，直到锁被释放为止。

```

2.如何加表锁

（1）MyISAM在执行查询语句（SELECT）前，会自动给涉及的所有表加读锁

（2）在执行更新操作（UPDATE、DELETE、INSERT等）前，会自动给涉及的表加写锁

3.MyISAM表锁优化建议

对于MyISAM存储引擎，虽然使用表级锁定在锁定实现的过程中比实现行级锁定或者页级锁所带来的附加成本都要小，锁定本身所消耗的资源也是最少。但是由
于并发处理能力不够。所以，在优化MyISAM存储引擎锁定问题的时候，应当需要尽可能让锁定的时间变短，然后就是让可能并发进行的操作尽可能的并发。

```text
（1）缩短锁定时间:

a)尽量减少大的复杂Query，将复杂Query分拆成几个小的Query分布进行；
b)尽可能的建立足够高效的索引，让数据检索更迅速；

（2）分离能并行的操作：

MyISAM存储引擎有一个控制是否打开Concurrent Insert功能的参数选项：concurrent_insert，可以设置为0，1或者2。三个值的具体说明如下：

concurrent_insert=2，无论MyISAM表中有没有空洞，都允许在表尾并发插入记录；

concurrent_insert=1，如果MyISAM表中没有空洞（即表的中间没有被删除的行），MyISAM允许在一个进程读表的同时，另一个进程从表尾插入记录。这也是MySQL的默认设置；

concurrent_insert=0，不允许并发插入。

```

###### 1.2.2 行锁的详解

行级锁定不是MySQL自己实现的锁定方式，而是由其他存储引擎InnoDB实现的。

1.MySQL行级锁的锁模式

```text
共享锁：当一个事务需要读取某个资源时加共享锁

排它锁：当一个事务需要修改删除某个资源时加排他锁

意向锁：为了让行级锁定和表级锁定共存，InnoDB也同样使用了意向锁（表级锁定）的概念，也就有了意向共享锁和意向排他锁这两种。意向锁是表级别的锁，
       用来说明事务稍后会对表中的数据行加哪种类型的锁(共享锁或独占锁)。当一个事务对表加了意向排他锁时，另外一个事务在加锁前就会通过该表的
       意向排他锁知道前面已经有事务在对该表进行独占操作，从而等待。
```
 ![image](https://github.com/jeremyke/PHPBlog/raw/master/Pictures/fc3d9ccb5e230ce9d260831588e78a8a_20181212202951237.png)

2.如何加表锁

（1）对于普通SELECT语句，InnoDB不会加任何锁；事务可以通过以下语句显示给记录加共享锁或排他锁。
```text
共享锁（S）：SELECT * FROM table_name WHERE ... LOCK IN SHARE MODE
排他锁（X)：SELECT * FROM table_name WHERE ... FOR UPDATE
```
（2）对于UPDATE、DELETE和INSERT语句，InnoDB会自动给涉及数据集加排他锁

（3）意向锁是InnoDB也是自动加的

3.innodb行锁优化

```text
a)尽可能让所有的数据检索都通过索引来完成，从而避免InnoDB因为无法通过索引键加锁而升级为表级锁定；

b)合理设计索引，让InnoDB在索引键上面加锁的时候尽可能准确，尽可能的缩小锁定范围，避免造成不必要的锁定而影响其他Query的执行；

c)尽可能减少基于范围的数据检索过滤条件，避免因为间隙锁带来的负面影响而锁定了不该锁定的记录；

d)尽量控制事务的大小，减少锁定的资源量和锁定时间长度；

e)在业务环境允许的情况下，尽量使用较低级别的事务隔离，以减少MySQL因为实现事务隔离级别所带来的附加成本。

```


#### 1.3 间隙锁

 当我们用范围条件而不是相等条件检索数据，并请求共享或排他锁时，InnoDB会给符合条件的已有数据记录的索引项加锁，并不会对对于键值在条件范围内
 但并不存在的记录加锁。这时候就需要间隙锁，来锁定在范围内，当不存在记录的行。

 **目的：**防止幻读，以满足相关隔离级别的要求。对于上面的例子，要是不使用间隙锁，如果其他事务插入了empid大于100的任何记录，那么本事务如
 果再次执行上述语句，就会发生幻读。


#### 1.4 死锁

当两个事务都需要获得对方持有的排他锁才能继续完成事务，这种循环锁等待就是典型的死锁。

- 解决方案：

InnoDB发现死锁之后，会计算出两个事务各自插入、更新或者删除的数据量来判定两个事务的大小。也就是说哪个事务所改变的记录条数越多，在死锁中就越
不会被回滚掉。当产生死锁的场景中涉及到不止InnoDB存储引擎的时候，InnoDB是没办法检测到该死锁的，这时候就只能通过锁定超时限制参数InnoDB_lock_wait_timeout来解决。

- 如何避免死锁：

```text
（1）在应用中，如果不同的程序会并发存取多个表，应尽量约定以相同的顺序来访问表，这样可以大大降低产生死锁的机会。

（2）在程序以批量方式处理数据的时候，如果事先对数据排序，保证每个线程按固定的顺序来处理记录，也可以大大降低出现死锁的可能。

（3）在事务中，如果要更新记录，应该直接申请足够级别的锁，即排他锁，而不应先申请共享锁，更新时再申请排他锁，因为当用户申请排他锁时，其他事务
可能又已经获得了相同记录的共享锁，从而造成锁冲突，甚至死锁。

（4）在可重复读隔离级别下，如果两个线程同时对相同条件记录用SELECT...FOR UPDATE加排他锁，在没有符合该条件记录情况下，两个线程都会加锁成功。
程序发现记录尚不存在，就试图插入一条新记录，如果两个线程都这么做，就会出现死锁。这种情况下，将隔离级别改成已提交读，就可避免问题。

（5）当隔离级别为已提交读时，如果两个线程都先执行SELECT...FOR UPDATE，判断是否存在符合条件的记录，如果没有，就插入记录。此时，只有一个线程
能插入成功，另一个线程会出现锁等待，当第1个线程提交后，第2个线程会因主键重出错，但虽然这个线程出错了，却会获得一个排他锁。这时如果有第3个线程
又来申请排他锁，也会出现死锁。对于这种情况，可以直接做插入操作，然后再捕获主键重异常，或者在遇到主键重错误时，总是执行ROLLBACK释放获得的排他锁。

```

 #### 1.5 简述MySQL的锁机制？

 ```text
    排它锁(也称独占锁、写锁或X锁)和共享锁(也称读锁或S锁)：
    (1)若sessionA获得某数据表的共享锁权限，那么任何session（包括sessionA）都能对该表进行读取，但是都不能修改该表。
    (2)若sessionA获得某数据表的排他锁权限，那么只有sessionA可以对该表进行读取或修改，其他session既不能读取也不能修改该表，更不能对该表加任何类型的锁，直到sessionA释放
    排它锁权限。加锁方式：lock tables tablename write/reade;释放锁：unlock tables;
    (3)若sessionA既获得某数据表的共享锁同时获取了该数据表的排它锁，那么只有sessionA可以对该表进行读取或修改，其他session既不能读取也不能修改该表。
    
    两种引擎对索引的支持的区别：
    （1）MyISAM:myisam只支持表锁,所有的锁机制是数据库自动加载的,在select时加读锁,在update,insert,delete写锁,读锁只兼容读锁,写锁排斥任何锁!也就是说当表存在写锁时,
     其他的操作只能排队等待了!
    （2）InnoDB:支持行锁,简单来说就是语句中使用到了索引,数据库就对对应的行加锁,如果没有用到索引,则将全表加锁.InnoDB在update,insert,delete给对应数据加排他锁,
    select通常情况下不加锁，意向锁是InnoDB自动加的，不需要用户干预.但可以通过select ...for update 和select ... lock in share mode显性加锁。
    
 ```


 ## 2 MySQL事务

 一个最小的不可再分的工作单元；通常一个事务对应一个完整的业务(例如银行账户转账业务，该业务就是一个最小的工作单元)。

 #### 2.1 ACID四种属性

  ```text
   (1)原子性：事务中的所有操作，要么全部完成，要么全部不完成，不可能停滞在中间某个环节。事务在执行过程中发生错误，会被回滚（Rollback）到事务开始前的状态，就像这个事务从来没有执行过一样。
   (2)一致性：事务只能把数据库从一个有效状态改变到另一个有效状态。有效状态指满足数据库所有的约束条件（定义的各种规则, 如主键, 外键, not null, unique, check等）。
   (3)隔离性：事务中的所有操作在未提交之前对其他事务是不可见的。
   (4)持久性：在事务完成以后，该事务所对数据库所作的更改便持久的保存在数据库之中，并不会被回滚。
  ```

 #### 2.2 事务隔离级别

 - (1) 未提交读(READ UNCOMMITTED)

  其他事务可以看到本事务没有提交的部分修改，因此会造成脏读的问题(读取到了其他事务未提交的部分,而之后该事务进行了回滚)。这个级别的性能没有足够大的优势,
  又有很多的问题,因此很少使用。此时select语句不加任何锁。此时并发最高，但会产生脏读。

 - (2) 已提交读(READ COMMITTED)

  其他事务只能读取到本事务已经提交的部分，这个隔离级别有不可重复读的问题,在同一个事务内的两次读取,拿到的结果竟然不一样,因为另外一个事务对数据进行了修改。
  普通select语句是快照读。update语句、delete语句、显示加锁的select语句（select … in share mode 或者 select … for update） 等，除了在外键约束检查和
  重复键检查时会封锁区间，其他情况都只使用行锁。

 - (3) 可重复读（REPEATABLE READ）

  通过使用了MVCC机制来实现可重复读，该隔离级别解决了上面不可重复读的问题,但是仍然有一个新问题,在同一个事务内的两次读取,无论其他事务有没有修改这行数据，拿到的结果都是一样的。
  但是会产生就幻读，当你读取id> 10的数据行时,对涉及到的所有行加上了读锁,此时另外一个事务新插入了一条id=11（id=11原先没有记录，不会加锁）的数据,
  那么进行本事务进行下一次的查询时会发现有一条id=11的数据,而上次的查询操作并没有获取到,再进行插入就会有主键冲突的问题，增加范围锁来解决这个问题。
  这个隔离级别也是Innodb存储引擎默认的隔离级别。普通select语句也是快照读。update语句、delete语句、显示加锁的select语句（select … in share 
  mode 或者 select … for update）则要分情况：
  ```text
  在唯一索引上使用唯一的查询条件，则使用记录锁。如: select * from user where id = 1;其中id建立了唯一索引。
  在唯一索引上使用 范围查询条件，则使用间隙锁与临键锁。如: select * from user where id >20;
  ```

 - (4) 可串行化(SERIALIZABLE)

 这是最高的隔离级别,可以解决上面提到的索引问题,因为他强制将所以的操作串行执行,这会导致并发性能极速下降,因此也不是很常用。此时所有select语
 句都会被隐式加锁：select … in share mode.


 #### 2.3 快照读、当前读

 MVCC并发控制中，读操作可以分成两类：快照读 (snapshot read)与当前读 (current read)。

 快照读，读取的是记录的可见版本 (有可能是历史版本)，不用加锁。

 当前读，读取的是记录的最新版本，并且，当前读返回的记录，都会加上锁，保证其他事务不会再并发修改这条记录。

 #### 2.4 事务的实现原理

 事务是通过redo日志和innodb的存储引擎日志缓冲（Innodb log buffer）来实现的。
 当开始一个事务的时候，会记录该事务的lsn(log sequence number)号;
 当事务执行时，会往InnoDB存储引擎的日志的日志缓存里面插入事务日志；
 当事务提交时，必须将存储引擎的日志缓冲写入磁盘（通过innodb_flush_log_at_trx_commit来控制）。
 也就是写数据前，需要先写日志。这种方式称为“预写日志方式”

 #### 2.5 简述MySQL的MVCC？

 加锁是一种控制并发的方式,但是加锁毕竟是一个比较消耗资源的操作,因此MySQL也实现了MVCC(Multi-Version Concurrency Control ),核心思想是给
 每一条数据加上两个版本号,一个是当前的数据版本号,一个是该数据的删除版本号。通过版本的控制,在一定程度上尚避免加锁也可以实现并发控制。

 在MySQL中,MVCC的大致工作原理如下:

 ```text
 select
 查询语句只会获取取符合下面两个条件的数据:一，数据版本号小于等于当前事务的版本号,这样可以保证查到的数据要么是之前就存在的,要么是本事务操作的。二，数据的删除版本号要么为空,要么大于事务当前的版本号，这样可以保证在此事务之前,该行数据没有被删除。
 
 insert
 插入数据，将数据的版本号设置为当前事务的版本号。
 
 delete
 删除数据，将删除行的删除版本号设置为当前事务的版本号.
 
 update
 对原数据进行删除操作,然后插入新数据,所以相当于上面两个操作的合集。
 ```

 #### 2.6 悲观锁与乐观锁？

悲观锁和乐观锁只是对数据验证的两种不同方式!只是一种概念。
```text
悲观锁:对数据的修改持以保守的态度,是利用数据库的锁机制,对修改的数据加排他锁!

乐观锁：相对悲观锁而言，乐观锁假设认为数据一般情况下不会造成冲突，所以在事务提交更新的时候，才会正式对数据的冲突与否进行检测, 较为盛行的方法有两种：

1）使用数据版本号记录机制实现，这是乐观锁最常用的一种实现方式。当读取数据时，将version字段的值一同读出，数据每更新一次，对此version值加一。当我们提交更新的时候，判断数据库表对应记录的当前版本信息与第一次取出来的version值进行比对，如果数据库表当前版本号与第一次取出来的version值相等，则予以更新，否则认为是过期数据。

2）在需要乐观锁控制的table中增加一个字段，名称无所谓，字段类型使用时间戳（timestamp）, 和上面的version类似，也是在更新提交的时候检查当前数据库中数据的时间戳和自己更新前取到的时间戳进行对比，如果一致则OK，否则就是版本冲突。

```
**总结：**

两种锁各有优缺点，不可认为一种好于另一种，像乐观锁适用于写比较少的情况下，即冲突真的很少发生的时候，这样可以省去了锁的开销，加大了系统的整个吞吐量。但如果经常产生冲突，上层应用会不断的进行retry，这样反倒是降低了性能，所以这种情况下用悲观锁就比较合适。另外，高并发情况下个人认为乐观锁要好于悲观锁，因为悲观锁的机制使得各个线程等待时间过长，极其影响效率，乐观锁可以在一定程度上提高并发度!


 ## 3. 索引
 >索引是存储引擎用于快速找到记录的一种数据结构，使得索引关键字和数据建立对应关系。

 #### 3.1 索引的实现原理？

索引目的是通过不断地缩小查找目的数据的范围来筛选出目的数据，同时把随机的事件变成顺序的事件，也就是说，有了这种索引机制，我们可以总是用同一种查找方式来锁定数据。数据库就是把数据分成若干段，精确定位查找范围，去除无效数据，提高查找速度。

 #### 3.2 索引的分类

 ```text
  主键索引：
         主键索引：primary key ：加速查找+约束（不为空且唯一）
  辅助索引：
         普通索引index :加速查找
         唯一索引：unique：加速查找+约束 （唯一）
         全文索引fulltext :用于搜索很长一篇文章的时候，效果最好。
         空间索引spatial :了解就好，几乎不用
         联合索引
             -primary key(id,name):联合主键索引
             -unique(id,name):联合唯一索引
             -index(id,name):联合普通索引
 ```

###### 3.2.1 B+Tree索引

查询数据是从索引的根节点开始，节点槽中存放了指向子节点的指针，存储引擎根据这些指针向下层查找。通过节点页的值和要查找的值可以找到合适的指针进入下一层子节点，最终找到对应的数据。叶子结点的指针指向的是被索引的数据，而不是其他节点页。

![image](https://github.com/jeremyke/PHPBlog/raw/master/Pictures/170010137312789.png)

- myisam实现B+Tree索引

![image](https://github.com/jeremyke/PHPBlog/raw/master/Pictures/5faccd0ae640ce696ef5fe0f2ec885b4_watermark,image_eXVuY2VzaGk=,t_100,g_se,x_0,y_0.jpg)

Myisam的叶子节点上存放了索引和指向被索引数据的地址。在myisam中，主键索引和辅助索引的结构是一样的，如上图，只是辅助索引的key可以重复。

- InnoDB实现B+Tree索引
![image](https://github.com/jeremyke/PHPBlog/raw/master/Pictures/f0117e257a771ea0c64b4d633497c26b_watermark,image_eXVuY2VzaGk=,t_100,g_se,x_0,y_0.jpg)

在InnoDB 中,表数据文件本身就是按 B+Tree 组织的一个索引结构,这棵树的叶点data 域保存了完整的数据记录。这个索引的 key 是数据表的主键,因此 InnoDB 表数据文件本身就是主键索引。
但是InnoDB的辅助索引data域存储相应记录主键的值而不是地址，查找的是有先找到数据对于的主键，在去回表找到主键对应的数据。

**聚簇索引与非聚簇索引**
```text
聚簇索引:在B-Tree基础上，把索引关键字和数据存放在一起的，数据行的所有数据全部存放在叶子节点上，紧紧相连，称之为聚簇索引。InnoDB的主键索引为聚簇结构。

非聚簇索引：MyISM 使用的是非聚簇索引, 非聚簇索引的两棵 B+树看上去没什么不同, 节点的结构完全一致只是存储的内容不同而已, 主键索引 B+树的节点存储了主键,
           辅助键索引B+树存储了辅助键。 表数据存储在独立的地方, 这两颗 B+树的叶子节点都使用一个地址指向真正的表数据, 对于表数据来说, 这两个键没有任何差别。
            由于索引树是独立的, 通过辅助键检索无需访问主键的索引树。
```

![image](https://github.com/jeremyke/PHPBlog/raw/master/Pictures/255c848eaaa5ff96947de694528be321_watermark,image_eXVuY2VzaGk=,t_100,g_se,x_0,y_0.jpg)


###### 3.2.2 哈希结构

对于每一行数据，根据存储引擎的所有列，生成哈希码，将所有的哈希码存放在索引中，同时哈希表中保存每个数据行的指针。精确匹配所有列才有效。

####  3.3索引对一下查询有效

-  全值匹配

  和索引中所有列进行匹配。

- 匹配最左前缀

  对于联合索引，只使用索引的第一列。

- 匹配列前缀

  只匹配某一列的值的开头部分。

- 匹配范围值

  匹配索引值在某个区间的查询。

- 精确匹配某一列并范围匹配另外一列

- 只访问索引的查询

  查询只需访问索引，无需访问数据行。

#### 3.4 索引对一下查询无效

- 如果不是按照索引的最左列开始查找，这无法使用索引。
- 不能跳过索引中的列。
- 如果查询中有某个列的范围查询，则其右边所有的列都无法使用到索引。

 #### 3.5 索引的优点

  ```text
（1）减少服务器需要扫描的数据量。
（2）帮助数据库避免排序和临时表。
（3）索引可以将随机I/O变为顺序I/O

  ```
 #### 3.6 索引的使用策略及其优化

 - 最左前缀原理与相关优化

 ```text
 （1）全列匹配：当按照索引中所有列进行精确匹配（这里精确匹配指“=”或“IN”匹配）时，索引可以被用到。即使顺序不一致，MySQL查询优化器会调整查询语句，
 使得were字段顺序和联合索引顺序一致。
 （2）最左前缀匹配：比如创建index a ('colus1','colus2','colus3')，相当于创建了索引('colus1')，（'colus1','colus2'），（'colus1','colus2','colus3'）
 （3）如果查询条件中含有函数或表达式，则MySQL不会为这列使用索引（虽然某些在数学意义上可以使用）。
 ```
 - 范围问题
 ```text
 （1）条件中出现这些符号或关键字：>、>=、<、<=、!= 、between...and...、like，对于主键索引，除了%置前的like,均会应用到索引。对于辅助索引，
  范围列可以用到索引（必须是最左前缀），比如（>,<）但是范围列后面的列无法用到索引。同时，索引最多用于一个范围列，因此如果查询条件中有两个范围列则无法全用到索引。
  当范围很小10%时，相当于多值精确匹配，会用到索引。
 （2）通配符写在最左边是无法命中索引的，一般写在后面。
 ```
 - 区分度高的列作为索引
 ```text
 尽量选择区分度高的列作为索引,区分度的公式是count(distinct col)/count(*)，表示字段不重复的比例，比例越大我们扫描的记录数越少，唯一键的区分度是1，
 而一些状态、性别字段可能在大数据面前区分度就是0。使用场景不同，这个值也很难确定，一般需要join的字段我们都要求是0.1以上，即平均1条扫描10条记录。
 ```
 - 前缀索引
 ```text
 列的前缀代替整个列作为索引key，当前缀长度合适时，可以做到既使得前缀索引的选择性接近全列索引，同时因为索引key变短而减少了索引文件的大小和维护开销。
 例如：SELECT count(DISTINCT(concat(first_name, n)))/count(*) AS Selectivity FROM employees.employees,当这个值很接近SELECT count(
 DISTINCT(concat(first_name)))/count(*) AS Selectivity FROM employees.employees就可以选择。
 ```
 - 索引不能参与计算
 ```text
索引列不能参与计算，保持列“干净”，比如from_unixtime(create_time) = ’2014-05-29’就不能使用到索引，原因很简单，b+树中存的都是数据表中的字段值，
但进行检索时，需要把所有元素都应用函数才能比较，显然成本太大。
 ```

 - 索引选择性
 ```text
 索引的代价：索引文件本身要消耗存储空间，同时索引会加重插入、删除和修改记录时的负担，另外，MySQL在运行时也要消耗资源维护索引，因此索引并不是越多越好。
 建索引依据：以2000行数据为分界线，大于2000需要建索引。当索引关键字取值范围比较少时，不宜建索引。
 ```


 ## 4. 日志

 #### 4.1 MySQL的日志分类
 ```text
 1.错误日志(error log)：记录mysql服务的启停时正确和错误的信息，还记录启动、停止、运行过程中的错误信息。
 2.查询日志(general log)：记录建立的客户端连接和执行（DQL）的语句。
 3.慢查询日志(slow log)：记录所有执行时间超过long_query_time的所有查询或不使用索引的查询。
 4.二进制日志(bin log)：记录所有更改数据的语句(DML,DDL)，可用于数据复制。
 5.中继日志(relay log)：主从复制时使用的日志。
 ```

 #### 4.2 错误日志

 错误日志用于记录MySQL服务进程mysqld在启动/关闭或运行过程中遇到的错误信息。

 - 开启日志

 在my.cnf配置文件中调整，注意，是在[mysqld_safe]或[mysqld]模块的下面进行配置:
 ```yaml
log-error = /data/mysql/error.err
 ```

- 查看日志

管理员可以使用命令轮询错误日志，例如可以按天轮询，具体方法如下：

```bash
cd /data/mysql/error.err
mv error.err error_$(date +%F).err
mysqladmin flush-logs
ls -l error.err
```

#### 4.3 慢查询日志

- 启动慢查询日志

```bash
mysql> set @@global.slow_query_log=on;
mysql> select sleep(10);//默认超时时长为10秒，所以进行一个10秒的查询
```
- 慢查询有关的变量:

```bash
long_query_time=10 # 指定慢查询超时时长(默认10秒)，超出此时长的属于慢查询
log_output={TABLE|FILE|NONE} # 定义一般查询日志和慢查询日志的输出格式，默认为file
log_slow_queries={yes|no}    # 是否启用慢查询日志，默认不启用
slow_query_log={1|ON|0|OFF}  # 也是是否启用慢查询日志，此变量和log_slow_queries修改一个另一个同时变化
slow_query_log_file=/mydata/data/hostname-slow.log  #默认路径为库文件目录下主机名加上-slow.log
log_queries_not_using_indexes=OFF # 查询没有使用索引的时候是否也记入慢查询日志
```
- 使用mysqldumpslow归类慢查询日志

根据mysqldumpslow给出的信息，可以进行单个sql针对性的分析：

```text
单次执行过长，可以使用explain进行分析，看是否使用索引，是都使用合适的索引。
执行次数过多，可以考虑减少重复执行，减轻单台机器压力（读写分离，读走从库之类的操作）。
锁持有时间过长，看是否有长事务，是否事务没有提交，是否锁竞争激烈，是否有死锁，是否加锁顺序不合理。
扫描行数过多，还是看索引。
返回用户行数过多，看是否能将sql拆分，返回合理的行数。
```

#### 4.4 二进制日志

二进制日志包含了引起或可能引起数据库改变(如delete语句但没有匹配行)的事件信息，但绝不会包括select和show这样的查询语句。语句以"事件"的形式保存，所以包含了时间、事件开始和结束位置等信息。

- 开启二进制日志

```bash
--log-bin=[on|off|file_name]
```

- 查看二进制日志

```bash
SHOW {BINARY | MASTER} LOGS      # 查看使用了哪些日志文件
SHOW BINLOG EVENTS [IN 'log_name'] [FROM pos]   # 查看日志中进行了哪些操作
SHOW MASTER STATUS         # 显式主服务器中的二进制日志信息
```

#### 4.5 MySQL的复制原理以及流程

 ![image](https://github.com/jeremyke/PHPBlog/raw/master/Pictures/4cec534924595826947f6957e010b801_master-slave.png)

 - 基本原理流程，3个线程以及之间的关联:
 ```text
1.binlog输出线程:每当有从库连接到主库的时候，主库都会创建一个线程然后发送binlog内容到从库。在从库里，当复制开始的时候，从库就会创建两个线程进行处理：
2.从库I/O线程:当START SLAVE语句在从库开始执行之后，从库创建一个I/O线程，该线程连接到主库并请求主库发送binlog日志更新到从库的中继日志relay log文件上。
3.从库的SQL线程:负责读取中继日志，解析出主服务器已经执行的数据更改并在从服务器中重放（Replay）。
 ```
 - 具体步骤
 ```text
步骤一：主库db的更新事件(update、insert、delete)被写到binlog
步骤二：从库发起连接，连接到主库
步骤三：此时主库创建一个binlog输出线程，把binlog的内容发送到从库
步骤四：从库启动之后，创建一个I/O线程，读取主库传过来的binlog内容并写入到relay log.
步骤五：还会创建一个SQL线程，从relay log里面读取内容，从Exec_Master_Log_Pos位置开始执行读取到的更新事件，将更新内容写入到slave的db.
 ```

#### 4.6 mysql数据实时同步到Elasticsearch

###### 4.6.1 mysqldump工具

mysqldump是一个对mysql数据库中的数据进行全量导出的一个工具。

```bash
#上述命令表示从远程数据库中导出database:webservice的所有数据，写入到dump.sql文件中，指定-F参数表示在导出数据后重新生成一个新的binlog日志文件以记录后续的所有数据操作。 
mysqldump -uelastic -p'密码' --host=主机 -F webservice > dump.sql
```

###### 4.6.2 go-mysql-elasticsearch

基本思想：

如果是第一次启动该程序，首先使用mysqldump工具对源mysql数据库进行一次全量同步，通过elasticsearch client执行操作写入数据到ES；
然后实现了一个mysql client,作为slave连接到源mysql,源mysql作为master会将所有数据的更新操作通过binlog event同步给slave， 
通过解析binlog event就可以获取到数据的更新内容，之后写入到ES.

使用限制：
  ```text
 记录mysql的binlog日志，再执行ES document api，将数据同步到ES集群中。
 mypipe同步数据到ES集群使用注意：
     1. mysql binlog必须是ROW模式
     2. 要同步的mysql数据表必须包含主键，否则直接忽略，这是因为如果数据表没有主键，UPDATE和DELETE操作就会因为在ES中找不到对应的document而无法进行同步
     3. 不支持程序运行过程中修改表结构
     4. 要赋予用于连接mysql的账户REPLICATION权限
        GRANT REPLICATION SLAVE, REPLICATION CLIENT ON *.* TO 'elastic'@'%' IDENTIFIED BY 'Elastic_123'
  ```


## 5. 高性能MySql

#### 5.1 读写分离

- 读写分离实现方式：

```text
1）配置多数据源；
2）使用mysql的proxy中间件代理工具；

第一种方式中，数据库和Application是有一定侵入性的，即我们的数据库更换时，application中的配置文件是需要手动修改的。
第二种方式中，我们可选择mysql proxy固定连接一个数据库，即使数据库地址更换也无需更换项目中的数据库连接配置。
同样，在开始配置实现MySQL读写分离之前，我们会遇到一个选型问题，那就是在诸多的MySQL的proxy中间件工具中，如mysql-proxy、atlas、cobar、
mycat、tddl、tinnydbrouter和mysql router等，我们该如何取舍呢？所以在择工具实现前，我们先对以上的proxy中间件做一个简单的优劣介绍，以便我们根据不同的场景选择。
```
- MyCat

**优点：**
```text
功能较丰富，对读写分离和分库分表都有支持；
易用，且对原有的应用系统侵入比较小，系统改造比较易于实现；
支持故障切换；
```
**缺点：**
```text
在整个系统中，MyCat作为一个单节点来路由其他数据库，在数据库比较多的情况下，MyCat本身的CPU性能压力会越来越大。因此，在生产系统中，MyCat不可避免的会需要一些高可用的手段；

同样，由于MyCat本身需要解析sql，也需要合并各个数据库返回的结果，本身CPU消耗会比较高，当达到一定临界点时，CPU可能会不堪重负。
```
 ![image](https://github.com/jeremyke/PHPBlog/raw/master/Pictures/1757111696117134.png)

 - 从库同步的延迟解决方案

 ```text
a. 架构层面：设置一主多从的模式。
  
b. 代码层面：拆分大事务，及时提交。
  
c. 数据库层面：检查表结构，保证每个表都有显式自增主键，并建立合适索引。
 ```


#### 5.2 分表

###### 5.2.1 垂直分表

垂直拆分是指数据表列的拆分，把一张列比较多的表拆分为多张表。 垂直分割一般用于拆分多字段和访问频率低的字段，分离冷热数据。适用于记录不是非常多的，但是字段却很多，这
样占用空间比较大，检索时需要执行大量的I/O，严重降低了性能，这个时候需要把大的自读那拆分到另一个表中，并且该表与源表时一对一关系。

###### 5.2.2 水平分表

水平拆分是指数据表行的拆分，表的行数超过500万行或者单表容量超过10GB时，查询就会变慢，这时可以把一张的表的数据拆成多张表来存放。水平分表尽可能使每张表的数据量相当，比较均匀。

- 水平分表的标准

```text
用户表可以根据用户的手机号段进行分割如user183、user150、user153、user189等，每个号段就是一张表

用户表也可以根据用户的id进行分割，加入分3张表user0,user1,user2，如果用户的id%3=0就查询user0表，如果用户的id%3=1就查询user1表

对于订单表可以按照订单的时间进行分表
```

#### 5.3 分区

分区是根据一定的规则，数据库把一个表分解成多个更小的、更容易管理的部分。就访问数据库应用而言，逻辑上就只有一个表或者一个索引，但实际上这个表
可能有N个物理分区对象组成，每个分区都是一个独立的对象，可以独立处理，可以作为表的一部分进行处理。分区对应用来说是完全透明的，不影响应用的业务逻辑。

分区有利于管理非常大的表，它采用分而治之的逻辑，分区引入了分区键的概念，分区键用于根据某个区间值(或者范围值)、特定值列表或者hash函数值执行数据的聚集，
让数据根据规则分布在不同的分区中，让一个大对象碧昂城一些小对象。MySQL分区即可以对数据进行分区也可以对索引进行分区。

- 分区类型

```text
range分区：基于一个给定的连续区间范围(区间要求连续并且不能重叠)，把数据分配到不同的分区
list分区：类似于range分区，区别在于list分区是居于枚举出的值列表分区，range是基于给定的连续区间范围分区
hash分区：基于给定的分区个数，把数据分配到不同的分区
key分区：类似于hash分区
```
更多内容可以参考：https://blog.csdn.net/vbirdbest/article/details/82461109



## 6. 通用模块

#### 6.1 存储引擎

|不同点|myisam|innodb|
|:----    |:---:|:-----:|
|（1）存储方式|数据和索引是分开存储的（3个文件（frm、MYD、MYI））|数据和索引是一起存储的（共享表空间存储和多表空间存储两种方式），2个存储文件（.ibd，.frm）|
|（2）存储顺序|插入顺序|主键顺序|
|（3）空间碎片的产生|会产生，需要定时清理（optimize table 表名）|不会产生|
|（4）事务和外键约束|不支持|支持|
|（5）锁级别|表锁|行锁|
|（5）读写插入|快速|更新删除快速|


 #### 6.2 MySQL中字段类型

 - varchar(50)中50的涵义

 首先明确：mysql中UTF-8编码,无论是一个汉字，英文，还是数字都占3个字节。这里50表示最多存放50个字符，varchar(50)和(200)存储hello所占空间一样，但后者在排序时会消耗更多内存，因为order by col采用fixed_length计算col长度(memory引擎也一样)

 - int（20）中20的涵义

 首先明确int类型只能占用4个字节的存储空间，这里20是指最大显示宽度，但是最大显示宽度为255。如果存储数据不够显示宽度，设置UNSIGNED ZEROFILL(无符号）就会在数据左侧用0来填充位数。

 #### 6.3 超键、候选键、主键、外键分别是什么？

 ```text
 1、超键：在关系中能唯一标识元组的属性集称为关系模式的超键。一个属性可以为作为一个超键，多个属性组合在一起也可以作为一个超键。超键包含候选键和主键。
 2、候选键：是最小超键，即没有冗余元素的超键。
 3、主键：数据库表中对储存数据对象予以唯一和完整标识的数据列或属性的组合。一个数据列只能有一个主键，且主键的取值不能缺失，即不能为空值（Null）。
 4、外键：在一个表中存在的另一个表的主键称此表的外键。
 ```

#### 6.4 ORM

  - 定义

  对象关系映射（Object Relational Mapping，简称ORM），是通过使用描述对象和数据库之间映射的元数据，将程序中的对象自动持久化到关系数据库中。它是
  为了解决面向对象与关系数据库存在的互不匹配的现象的技术。ORM在业务逻辑层和数据库层之间充当了桥梁的作用。

#### 6.5 PHP中MySQL、MySQLi和PDO的用法和区别

 - PHP的MySQL扩展

 设计开发允许PHP应用与MySQL数据库交互的早期扩展，mysql扩展提供了一个面向过程的接口，这里不做过多讨论。

 ```php
 <?php
 $conn= @mysql_connect("localhost", "root", "") or die("数据库连接错误");
 mysql_select_db("bbs", $conn);
 mysql_query("set names 'utf8'");
 echo "数据库连接成功";
 ?>
 ```
 **说明**
 （1）允许 PHP 应用与 MySQL 数据库交互的早期扩展
 （2）提供了一个面向过程的接口，不支持后期的一些特性

 - MySQLi扩展

 mysqli扩展有一系列的优势，相对于mysql扩展的提升主要有：面向对象接口、 prepared语句支持、多语句执行支持、事务支持、增强的调试能力、嵌入式服务支持。
 ```php
 <?php
 $conn= mysqli_connect('localhost', 'root', '', 'bbs');
 if(!$conn){
     die("数据库连接错误". mysqli_connect_error());
 }else{
     echo"数据库连接成功";
 }
 ?>
 ```

 **说明**
 （1）面向对象接口
 （2）prepared 语句支持
 （3）多语句执行支持
 （4）事务支持
 （5）增强的调试能力 

 - PDO

 PHP数据对象，是PHP应用中的一个数据库抽象层规范。PDO提供了一个统一的API接口可以，使得你的PHP应用不去关心具体要连接的数据库服务器系统类型。
 也就是说，如果你使用PDO的API，可以在任何需要的时候无缝切换数据库服务器。

 ```php
 <?php
 try{
     $pdo=new  pdo("mysql:host=localhost;dbname=bbs","root","");
 }catch(PDDException $e){
     echo"数据库连接错误";
 }
 echo"数据库连接成功";
 ?>
 ```
  **说明**
  （1）PHP 应用中的一个数据库抽象层规范
  （2）PDO 提供一个统一的 API 接口，无须关心数据库类型
  （3）使用标准的 PDO API，可以快速无缝切换数据库


## 7. 开放性问题

#### 7.1 MySQL数据库cpu飙升到500%的话他怎么处理？

 ```text
 1、列出所有进程  show processlist,观察所有进程 ,多秒没有状态变化的(干掉)
 2、查看超时日志或者错误日志 ,一般会是查询以及大批量的插入会导致cpu与i/o上涨,当然不排除网络状态突然断了,导致一个请求服务
 器只接受到一半，比如where子句或分页子句没有发送。
 ```
#### 7.1 MySQL优化？

###### 7.1.1 字段设计

- 尽可能选择小的数据类型：比如ip用int而不是varchar,金额使用decimal或者int，char代替varchar。

- 尽可能使用not null：非空比空处理效率快，空会额外增加一个字节的空间。

- 单表尽量少于30个字段。

###### 7.1.2 引擎选择

- innodb擅长更新和删除，同时也支持事务，外键等高级功能。

- myisam擅长查询及插入。

###### 7.1.3 索引优化

- a、对WHERE子句中频繁使用的建立索引；

- b、尽可能使用唯一索引，重复值越少，索引效果越强；

- c、尽量使用短索引，如果char(255)太大，应该给它指定一个前缀长度，大部分情况下前10位或20位值基本是唯一的，那么就不要对整个列进行索引；

- d、充分利用左前缀，对WHERE语句如果有AND并列，只能识别一个索引(获取记录最少的那个)，这时可以将AND的字段加复合索引;

- e、不要过度索引。索引越多，占用空间越大，反而性能变慢；

- f、应尽量使用到索引覆盖，避免回表。

###### 7.1.4 sql语句优化

- a、尽量不要使用select（*）

- b、where语句的通配符尽量不要放在前面，且最好不要给字段加运算。

- c、尽量不要使用limit，因为limit的操作是在内存中执行的，实际上IO以及查询出来所有复合条件的数据。

- d、尽量少用多表连接，将复杂的sql拆分多次执行。



###### 7.1.5 缓存

- 开启缓存;query_cache_type设置为ON

###### 7.1.6 分区分表

###### 7.1.7 服务器架构

- 采用一主多从，读写分离的架构。

 


