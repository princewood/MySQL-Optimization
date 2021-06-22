# MySQL优化

作者：zhuqz
链接：https://www.zhihu.com/question/19719997/answer/81930332
来源：知乎

第一优化你的sql和索引；第二加缓存，memcached,redis；第三以上都做了后，还是慢，就做主从复制或主主复制，读写分离，可以在应用层做，效率高，也可以用三方工具，第三方工具推荐360的atlas,其它的要么效率不高，要么没人维护；第四如果以上都做了还是慢，不要想着去做切分，mysql自带分区表，先试试这个，对你的应用是透明的，无需更改代码,但是sql语句是需要针对分区表做优化的，sql条件中要带上分区条件的列，从而使查询定位到少量的分区上，否则就会扫描全部分区，另外分区表还有一些坑，在这里就不多说了；第五如果以上都做了，那就先做垂直拆分，其实就是根据你模块的耦合度，将一个大的系统分为多个小的系统，也就是分布式系统；第六才是水平切分，针对数据量大的表，这一步最麻烦，最能考验技术水平，要选择一个合理的sharding key,为了有好的查询效率，表结构也要改动，做一定的冗余，应用也要改，sql中尽量带sharding key，将数据定位到限定的表上去查，而不是扫描全部的表；mysql数据库一般都是按照这个步骤去演化的，成本也是由低到高；有人也许要说第一步优化sql和索引这还用说吗？的确，大家都知道，但是很多情况下，这一步做的并不到位，甚至有的只做了根据sql去建索引，根本没对sql优化（中枪了没？），除了最简单的增删改查外，想实现一个查询，可以写出很多种查询语句，不同的语句，根据你选择的引擎、表中数据的分布情况、索引情况、数据库优化策略、查询中的锁策略等因素，最终查询的效率相差很大；优化要从整体去考虑，有时你优化一条语句后，其它查询反而效率被降低了，所以要取一个平衡点；即使精通mysql的话，除了纯技术面优化，还要根据业务面去优化sql语句，这样才能达到最优效果；你敢说你的sql和索引已经是最优了吗?再说一下不同引擎的优化，myisam读的效果好，写的效率差，这和它数据存储格式，索引的指针和锁的策略有关的，它的数据是顺序存储的（innodb数据存储方式是聚簇索引），他的索引btree上的节点是一个指向数据物理位置的指针，所以查找起来很快，（innodb索引节点存的则是数据的主键，所以需要根据主键二次查找）；myisam锁是表锁，只有读读之间是并发的，写写之间和读写之间（读和插入之间是可以并发的，去设置concurrent_insert参数，定期执行表优化操作，更新操作就没有办法了）是串行的，所以写起来慢，并且默认的写优先级比读优先级高，高到写操作来了后，可以马上插入到读操作前面去，如果批量写，会导致读请求饿死，所以要设置读写优先级或设置多少写操作后执行读操作的策略;myisam不要使用查询时间太长的sql，如果策略使用不当，也会导致写饿死，所以尽量去拆分查询效率低的sql,innodb一般都是行锁，这个一般指的是sql用到索引的时候，行锁是加在索引上的，不是加在数据记录上的，如果sql没有用到索引，仍然会锁定表,mysql的读写之间是可以并发的，普通的select是不需要锁的，当查询的记录遇到锁时，用的是一致性的非锁定快照读，也就是根据数据库隔离级别策略，会去读被锁定行的快照，其它更新或加锁读语句用的是当前读，读取原始行；因为普通读与写不冲突，所以innodb不会出现读写饿死的情况，又因为在使用索引的时候用的是行锁，锁的粒度小，竞争相同锁的情况就少，就增加了并发处理，所以并发读写的效率还是很优秀的，问题在于索引查询后的根据主键的二次查找导致效率低；ps:很奇怪，为什innodb的索引叶子节点存的是主键而不是像mysism一样存数据的物理地址指针吗？如果存的是物理地址指针不就不需要二次查找了吗，这也是我开始的疑惑，根据mysism和innodb数据存储方式的差异去想，你就会明白了，我就不费口舌了！所以innodb为了避免二次查找可以使用索引覆盖技术，无法使用索引覆盖的，再延伸一下就是基于索引覆盖实现延迟关联；不知道什么是索引覆盖的，建议你无论如何都要弄清楚它是怎么回事！尽你所能去优化你的sql吧！说它成本低，却又是一项费时费力的活，需要在技术与业务都熟悉的情况下，用心去优化才能做到最优，优化后的效果也是立竿见影的！



作者：互联网编程
链接：https://www.zhihu.com/question/19719997/answer/549041957
来源：知乎

# 问题概述
使用阿里云rds for MySQL数据库（就是MySQL5.6版本），有个用户上网记录表6个月的数据量近2000万，保留最近一年的数据量达到4000万，查询速度极慢，日常卡死。严重影响业务。问题前提：老系统，当时设计系统的人大概是大学没毕业，表设计和sql语句写的不仅仅是垃圾，简直无法直视。原开发人员都已离职，到我来维护，这就是传说中的维护不了就跑路，然后我就是掉坑的那个！！！我尝试解决该问题，so，有个这个日志。  方案概述 方案一：优化现有mysql数据库。优点：不影响现有业务，源程序不需要修改代码，成本最低。缺点：有优化瓶颈，数据量过亿就玩完了。 方案二：升级数据库类型，换一种100%兼容mysql的数据库。优点：不影响现有业务，源程序不需要修改代码，你几乎不需要做任何操作就能提升数据库性能，缺点：多花钱 方案三：一步到位，大数据解决方案，更换newsql/nosql数据库。优点：没有数据容量瓶颈，缺点：需要修改源程序代码，影响业务，总成本最高。以上三种方案，按顺序使用即可，数据量在亿级别一下的没必要换nosql，开发成本太高。三种方案我都试了一遍，而且都形成了落地解决方案。该过程心中慰问跑路的那几个开发者一万遍 :) 方案一详细说明：优化现有mysql数据库跟阿里云数据库大佬电话沟通 and  Google解决方案  and 问群里大佬，总结如下（都是精华）： 1.数据库设计和表创建时就要考虑性能 2.sql的编写需要注意优化 3.分区 4.分表 5.分库 1.数据库设计和表创建时就要考虑性能mysql数据库本身高度灵活，造成性能不足，严重依赖开发人员能力。也就是说开发人员能力高，则mysql性能高。这也是很多关系型数据库的通病，所以公司的dba通常工资巨高。 设计表时要注意： 表字段避免null值出现，null值很难查询优化且占用额外的索引空间，推荐默认数字0代替null。 尽量使用INT而非BIGINT，如果非负则加上UNSIGNED（这样数值容量会扩大一倍），当然能使用TINYINT、SMALLINT、MEDIUM_INT更好。 使用枚举或整数代替字符串类型 尽量使用TIMESTAMP而非DATETIME 单表不要有太多字段，建议在20以内 用整型来存IP 索引 索引并不是越多越好，要根据查询有针对性的创建，考虑在WHERE和ORDER BY命令上涉及的列建立索引，可根据EXPLAIN来查看是否用了索引还是全表扫描 应尽量避免在WHERE子句中对字段进行NULL值判断，否则将导致引擎放弃使用索引而进行全表扫描 值分布很稀少的字段不适合建索引，例如"性别"这种只有两三个值的字段 字符字段只建前缀索引 字符字段最好不要做主键 不用外键，由程序保证约束 尽量不用UNIQUE，由程序保证约束 使用多列索引时主意顺序和查询条件保持一致，同时删除不必要的单列索引 简言之就是使用合适的数据类型，选择合适的索引1.选择合适的数据类型（1）使用可存下数据的最小的数据类型，整型 < date,time < char,varchar < blob（2）使用简单的数据类型，整型比字符处理开销更小，因为字符串的比较更复杂。如，int类型存储时间类型，bigint类型转ip函数（3）使用合理的字段属性长度，固定长度的表会更快。使用enum、char而不是varchar（4）尽可能使用not null定义字段（5）尽量少用text，非用不可最好分表 2.选择合适的索引列（1）查询频繁的列，在where，group by，order by，on从句中出现的列（2）where条件中<，<=，=，>，>=，between，in，以及like 字符串+通配符（%）出现的列（3）长度小的列，索引字段越小越好，因为数据库的存储单位是页，一页中能存下的数据越多越好（4）离散度大（不同的值多）的列，放在联合索引前面。查看离散度，通过统计不同的列值来实现，count越大，离散程度越高： 原开发人员已经跑路，该表早已建立，我无法修改，故：该措辞无法执行，放弃！ 2.sql的编写需要注意优化 使用limit对查询结果的记录进行限定 避免`select *`，将需要查找的字段列出来 使用连接（join）来代替子查询 拆分大的delete或insert语句 可通过开启慢查询日志来找出较慢的SQL 不做列运算：SELECT id WHERE age + 1 = 10，任何对列的操作都将导致表扫描，它包括数据库教程函数、计算表达式等等，查询时要尽可能将操作移至等号右边 sql语句尽可能简单：一条sql只能在一个cpu运算；大语句拆小语句，减少锁时间；一条大sql可以堵死整个库 OR改写成IN：OR的效率是n级别，IN的效率是log(n)级别，in的个数建议控制在200以内 不用函数和触发器，在应用程序实现 避免%xxx式查询 少用JOIN 使用同类型进行比较，比如用'123'和'123'比，123和123比 尽量避免在WHERE子句中使用!=或<>操作符，否则将引擎放弃使用索引而进行全表扫描 对于连续数值，使用BETWEEN不用IN：SELECT id FROM t WHERE num BETWEEN 1 AND 5 列表数据不要拿全表，要使用LIMIT来分页，每页数量也不要太大 原开发人员已经跑路，程序已经完成上线，我无法修改sql，故：该措辞无法执行，放弃！引擎引擎目前广泛使用的是MyISAM和InnoDB两种引擎：1. MyISAMMyISAM引擎是MySQL 5.1及之前版本的默认引擎，它的特点是： 不支持行锁，读取时对需要读到的所有表加锁，写入时则对表加排它锁 不支持事务 不支持外键 不支持崩溃后的安全恢复 在表有读取查询的同时，支持往表中插入新纪录 支持BLOB和TEXT的前500个字符索引，支持全文索引 支持延迟更新索引，极大提升写入性能 对于不会进行修改的表，支持压缩表，极大减少磁盘空间占用2. InnoDBInnoDB在MySQL 5.5后成为默认索引，它的特点是： 支持行锁，采用MVCC来支持高并发 支持事务 支持外键 支持崩溃后的安全恢复 不支持全文索引总体来讲，MyISAM适合SELECT密集型的表，而InnoDB适合INSERT和UPDATE密集型的表 MyISAM速度可能超快，占用存储空间也小，但是程序要求事务支持，故InnoDB是必须的，故该方案无法执行，放弃！ 3.分区MySQL在5.1版引入的分区是一种简单的水平拆分，用户需要在建表的时候加上分区参数，对应用是透明的无需修改代码对用户来说，分区表是一个独立的逻辑表，但是底层由多个物理子表组成，实现分区的代码实际上是通过对一组底层表的对象封装，但对SQL层来说是一个完全封装底层的黑盒子。MySQL实现分区的方式也意味着索引也是按照分区的子表定义，没有全局索引用户的SQL语句是需要针对分区表做优化，SQL条件中要带上分区条件的列，从而使查询定位到少量的分区上，否则就会扫描全部分区，可以通过EXPLAIN PARTITIONS来查看某条SQL语句会落在那些分区上，从而进行SQL优化，我测试，查询时不带分区条件的列，也会提高速度，故该措施值得一试。 分区的好处是： 可以让单表存储更多的数据 分区表的数据更容易维护，可以通过清楚整个分区批量删除大量数据，也可以增加新的分区来支持新插入的数据。另外，还可以对一个独立分区进行优化、检查、修复等操作 部分查询能够从查询条件确定只落在少数分区上，速度会很快 分区表的数据还可以分布在不同的物理设备上，从而搞笑利用多个硬件设备 可以使用分区表赖避免某些特殊瓶颈，例如InnoDB单个索引的互斥访问、ext3文件系统的inode锁竞争 可以备份和恢复单个分区 分区的限制和缺点： 一个表最多只能有1024个分区 如果分区字段中有主键或者唯一索引的列，那么所有主键列和唯一索引列都必须包含进来 分区表无法使用外键约束 NULL值会使分区过滤无效 所有分区必须使用相同的存储引擎 分区的类型： RANGE分区：基于属于一个给定连续区间的列值，把多行分配给分区 LIST分区：类似于按RANGE分区，区别在于LIST分区是基于列值匹配一个离散值集合中的某个值来进行选择 HASH分区：基于用户定义的表达式的返回值来进行选择的分区，该表达式使用将要插入到表中的这些行的列值进行计算。这个函数可以包含MySQL中有效的、产生非负整数值的任何表达式 KEY分区：类似于按HASH分区，区别在于KEY分区只支持计算一列或多列，且MySQL服务器提供其自身的哈希函数。必须有一列或多列包含整数值具体关于mysql分区的概念请自行google或查询官方文档，我这里只是抛砖引玉了。 我首先根据月份把上网记录表RANGE分区了12份，查询效率提高6倍左右，效果不明显，故：换id为HASH分区，分了64个分区，查询速度提升显著。问题解决！ 结果如下：PARTITION BY HASH (id)PARTITIONS 64 select count(*) from readroom_website; --11901336行记录/* 受影响行数: 0  已找到记录: 1  警告: 0  持续时间 1 查询: `5.734 sec. */`   `select * from readroom_website where month(accesstime) =11 limit 10;/*` 受影响行数: 0  已找到记录: 10  警告: 0  持续时间 1 查询: `0.719 sec. */` 4.分表分表就是把一张大表，按照如上过程都优化了，还是查询卡死，那就把这个表分成多张表，把一次查询分成多次查询，然后把结果组合返回给用户。分表分为垂直拆分和水平拆分，通常以某个字段做拆分项。比如以id字段拆分为100张表： 表名为  tableName_id%100但：分表需要修改源程序代码，会给开发带来大量工作，极大的增加了开发成本，故：只适合在开发初期就考虑到了大量数据存在，做好了分表处理，不适合应用上线了再做修改，成本太高！！！而且选择这个方案，都不如选择我提供的第二第三个方案的成本低！故不建议采用。 5.分库把一个数据库分成多个，建议做个读写分离就行了，真正的做分库也会带来大量的开发成本，得不偿失！不推荐使用。 方案二详细说明：升级数据库，换一个100%兼容mysql的数据库mysql性能不行，那就换个。为保证源程序代码不修改，保证现有业务平稳迁移，故需要换一个100%兼容mysql的数据库。1. 开源选择 tiDB  pingcap/tidb Cubrid  Open Source Database With Enterprise Features开源数据库会带来大量的运维成本且其工业品质和MySQL尚有差距，有很多坑要踩，如果你公司要求必须自建数据库，那么选择该类型产品。2. 云数据选择 阿里云POLARDB 云数据库POLARDB_高吞吐在线事务处理_关系型云数据库_价格_购买 - 阿里云 官方介绍语：POLARDB 是阿里云自研的下一代关系型分布式云原生数据库，100%兼容MySQL，存储容量最高可达 100T，性能最高提升至 MySQL 的 6 倍。POLARDB 既融合了商业数据库稳定、可靠、高性能的特征，又具有开源数据库简单、可扩展、持续迭代的优势，而成本只需商用数据库的 1/10。我开通测试了一下，支持免费mysql的数据迁移，无操作成本，性能提升在10倍左右，价格跟rds相差不多，是个很好的备选解决方案！ 阿里云OcenanBase  淘宝使用的，扛得住双十一，性能卓著，但是在公测中，我无法尝试，但值得期待 阿里云HybridDB for MySQL (原PetaData)  云数据库HybridDB for MySQL_产品详情_阿里云 官方介绍：云数据库HybridDB for MySQL （原名PetaData）是同时支持海量数据在线事务（OLTP）和在线分析（OLAP）的HTAP（Hybrid Transaction/Analytical Processing）关系型数据库。我也测试了一下，是一个olap和oltp兼容的解决方案，但是价格太高，每小时高达10块钱，用来做存储太浪费了，适合存储和分析一起用的业务。 腾讯云DCDB分布式数据库 - 腾讯云 官方介绍：DCDB又名TDSQL，一种兼容MySQL协议和语法，支持自动水平拆分的高性能分布式数据库——即业务显示为完整的逻辑表，数据却均匀的拆分到多个分片中；每个分片默认采用主备架构，提供灾备、恢复、监控、不停机扩容等全套解决方案，适用于TB或PB级的海量数据场景。腾讯的我不喜欢用，不多说。原因是出了问题找不到人，线上问题无法解决头疼！但是他价格便宜，适合超小公司，玩玩。 方案三详细说明：去掉mysql，换大数据引擎处理数据数据量过亿了，没得选了，只能上大数据了。1. 开源解决方案hadoop家族。hbase/hive怼上就是了。但是有很高的运维成本，一般公司是玩不起的，没十万投入是不会有很好的产出的！2.云解决方案这个就比较多了，也是一种未来趋势，大数据由专业的公司提供专业的服务，小公司或个人购买服务，大数据就像水/电等公共设施一样，存在于社会的方方面面。国内做的最好的当属阿里云。我选择了阿里云的MaxCompute配合DataWorks，使用超级舒服，按量付费，成本极低。MaxCompute可以理解为开源的Hive，提供sql/mapreduce/ai算法/python脚本/shell脚本等方式操作数据，数据以表格的形式展现，以分布式方式存储，采用定时任务和批处理的方式处理数据。DataWorks提供了一种工作流的方式管理你的数据处理任务和调度监控。当然你也可以选择阿里云hbase等其他产品，我这里主要是离线处理，故选择MaxCompute，基本都是图形界面操作，大概写了300行sql，费用不超过100块钱就解决了数据处理问题。

![image](https://user-images.githubusercontent.com/58221386/122941455-b2d6fb80-d375-11eb-9c61-5e9ce542b608.png)
