这里主要讲解一下，在MySQL开发中一些常用的设计规约，如 数据表结构设计规范和索引设计规范。

# 基本规范

基本要求如下：

> 1、所有表必须使用Innodb存储引擎
>
> 2、数据库和表的字符集统一使用UTF8
>
> 3、所有表和字段都需要添加注释
>
> 4、尽量控制单表数据量的大小，建议控制在500万以内
>
> 5、谨慎使用MySQL分区表
>
> 6、尽量做到冷热数据分离，减小表的宽度
>
> 7、禁止在表中建立预留字段
>
> 8、禁止在数据库中存储图片，文件等大的二进制数据

以下便一一进行分析

**1、所有表必须使用Innodb存储引擎**

没有特殊要求（即Innodb无法满足的功能如：列存储，存储空间数据等）的情况下，所有表必须使用Innodb存储引擎（mysql5.5之前默认使用Myisam，5.6以后默认的为Innodb）Innodb 支持事务，支持行级锁，更好的恢复性，高并发下性能更好

**2、数据库和表的字符集统一使用UTF8**

兼容性更好，统一字符集可以避免由于字符集转换产生的乱码，不同的字符集进行比较前需要进行转换会造成索引失效

**3、所有表和字段都需要添加注释**

使用comment从句添加表和列的备注 从一开始就进行数据字典的维护

**4、尽量控制单表数据量的大小，建议控制在500万以内**

500万并不是MySQL数据库的限制，过大会造成修改表结构，备份，恢复都会有很大的问题可以用历史数据归档（应用于日志数据），分库分表（应用于业务数据）等手段来控制数据量大小

**5、谨慎使用MySQL分区表**

分区表在物理上表现为多个文件，在逻辑上表现为一个表zai选择分区键，跨分区查询效率可能更低，建议采用物理分表的方式管理大数据

**6、尽量做到冷热数据分离，减小表的宽度**

MySQL限制每个表最多存储4096列，并且每一行数据的大小不能超过65535字节

表越宽，把表装载进内存缓冲池时所占用的内存也就越大,也会消耗更多的IO

减少磁盘IO,保证热数据的内存缓存命中率；

更有效的利用缓存，避免读入无用的冷数据 经常一起使用的列放到一个表中（避免更多的关联操作）

**7、禁止在表中建立预留字段**

预留字段的命名很难做到见名识义 预留字段无法确认存储的数据类型，所以无法选择合适的类型 对预留字段类型的修改，会对表进行锁定

**8、禁止在数据库中存储图片，文件等大的二进制数据**

通常文件很大，会短时间内造成数据量快速增长，数据库进行数据库读取时，通常会进行大量的随机IO操作，文件很大时，IO操作很耗时 通常存储于文件服务器，数据库只存储文件地址信息



# 数据库字段设计

基本要求如下：

* 优先选择符合存储需要的最小的数据类型
* 主键列最好定义为自增主键
* 同财务相关的金额类数据必须使用DECIMAL类型
* 使用TIMESTAMP（4个字节）而非DATETIME （8个字节）
* 尽可能把所有列定义为NOT NULL
* 避免使用TEXT、BLOB数据类型
* 避免使用ENUM类型

尽可能单表不要有太多字段，建议在20以内，下面一一进行分析：

**1、最小数据类型**

因为列的字段越大，建立索引时所需要的空间也就越大，这样一页中所能存储的索引节点的数量也就越少也越少，在遍历时所需要的IO次数也就越多， 索引的性能也就越差

**优化方案**

* **尽量使用最小数字类型，如果非负则加上UNSIGNED，**特别对于非负型的数据（如自增ID、整型IP）因为：无符号相对于有符号可以多出一倍的存储空间。TINYINT< SMALLINT< MEDIUM_INT<  INT < BIGINT
* **尽可能使用 varchar代替 char，**因为首先变长字段存储空间小，可以节省存储空间，其次对于查询来说，在一个相对较小的字段内搜索效率显然要高些。当然列值确定为固定长度，这优先使用char类型
* **尽可能使用整数代替字符串类型**，如1，性别：可以使用 tinyint，它比 char 更合适；如2：将IP地址转换成整形数据，IP处理函数inet_aton()和inet_ntoa()；插入数据前，先用inet_aton把ip地址转为整型，可以节省空间。显示数据时，使用inet_ntoa把整型的ip地址转为地址显示即可。

**char和varchar区别：**

char是属于固定长度的字符类型，而varchar是属于可变长度的字符类型，

因为其长度固定，方便程序的存储与查找，char的存取数度还是要比varchar要快得多，但是char也为此付出的是空间的代价，这属于时间换空间的做法，varchar 则相反，属于空间换时间

**2、主键列最好定义为自增主键**

>  InnoDB使用聚集索引，数据记录本身被存于主索引（一颗B+Tree）的叶子节点上。
>
> 这就要求同一个叶子节点内（大小为一个内存页或磁盘页）的各条数据记录按主键顺序存放，因此每当有一条新的记录插入时，MySQL会根据其主键将其插入适当的节点和位置，如果页面达到装载因子（InnoDB默认为15/16），则开辟一个新的页（节点）。

**InnoDB为什么推荐使用自增ID作为主键？**

  **答：自增ID可以保证每次插入时B+索引是从右边扩展的，可以避免B+树和频繁合并和分裂（对比使用UUID）。如果使用字符串主键和随机主键，会使得数据随机插入，效率比较差。**

**自增主键优缺点**

优点：

1. 数据库自动编号，速度快，而且是增量增长，按顺序存放，对于检索非常有利；
2. 数字型，占用空间小，易排序，在程序中传递也方便；

缺点：

1. 因为自动增长，导入数据时很难保证原系统的ID不发生主键冲突
2. 自增主键插入式顺序插入，存在并发插入时导致间隙锁竞争

**UUID主键优缺点**

优点：

1. 能够保证独立性，程序可以在不同的数据库间迁移，效果不受影响。 
2. 全局的唯一性，不仅是表独立的，而且是库独立的，这点在你想切分数据库的时候尤为重要。

缺点：

1. UUID占空间大， 如果你建的索引越多， 影响越严重，且会有页分裂和碎片存在导致。
2. UUID之间比较大小相对数字慢不少， 影响查询速度
3. 影响插入速度， 并且造成硬盘使用率低

**3、同财务相关的金额类数据必须使用decimal类型**

float,double 是非精准浮点类型，要想精准需要使用 decimal

**4、使用TIMESTAMP（4个字节）而非DATETIME （8个字节）**

TIMESTAMP只使用DATETIME一半的存储空间、并且会根据时区变化，缺点TIMESTAMP的时间范围要小

TIMESTAMP 存储的时间范围 1970-01-01 00:00:01 ~ 2038-01-19-03:14:07。

TIMESTAMP 占用4字节和INT相同，但比INT可读性高；超出TIMESTAMP取值范围的使用DATETIME类型存储。

经常会有人用字符串存储日期型的数据（不正确的做法），

缺点1：无法用日期函数进行计算和比较；

缺点2：用字符串存储日期要占用更多的空间

**5、尽可能把所有列定义为NOT NULL**

注意：把 NULL 改成 NOT NULL 对索引的性能并没有明显的提升。避免使用 NULL 的目的，是便于代码的可读性和可维护性。

同时也便于避免下文即将出现的一些稀奇古怪的错误，如：

* NOT IN、!= 等负向条件查询在有 NULL 值的情况下返回非空行的结果集
* 使用 concat 函数拼接时，首先要对各个字段进行非 NULL 判断，否则只要任何一个字段为空都会造成拼接的结果为 NULL
* 当用count函数进行统计时，NULL 列不会计入统计
* 查询空行数据，用 is NULL
* NULL 列需要更多的存储空间，一般需要一个额外的字节作为判断是否为 NULL 的标志位。
* IS NULL 或者 IS NOT NULL 索引会失效

**6、TEXT、BLOB数据类型**

Mysql内存临时表不支持TEXT、BLOB这样的大数据类型，如果查询中包含这样的数据，在排序等操作时，就不能使用内存临时表，必须使用磁盘临时表进行。

对于这种数据，Mysql还是要进行二次查询，会使sql性能变得很差，但是不是说一定不能使用这样的数据类型。

因为MySQL对索引字段长度是有限制的，所以TEXT类型只能使用前缀索引，并且TEXT列上是不能有默认值的。

**7、避免使用ENUM类型**

修改ENUM值需要使用ALTER语句；**ENUM类型的ORDER BY操作效率低，需要额外操作**；禁止使用数值作为ENUM的枚举值



# 索引设计规范

创建索引时，最适合索引的列是出现在where 子句中的列，或连接子句中指定的列，而不是select 关键字后的列。索引列基数越大效果越好，但如记录性别的列，只含M和F这种情况，就没必要使用索引

基本要求如下：

* 限制每张表上的索引数量，建议单张表索引不超过5个
* 禁止给表中的每一列都建立单独的索引
* 每个Innodb表必须有个主键
* 如何选择索引列的顺序
* 优先考虑覆盖索引
* 避免使用外键约束

**1、限制每张表上的索引数量**

索引并不是越多越好！索引可以提高效率同样可以降低效率。

因为mysql优化器在选择如何优化查询时，会根据统一信息，对每一个可以用到的索引来进行评估，以生成出一个最好的执行计划，如果同时有很多个索引都可以用于查询，就会增加mysql优化器生成执行计划的时间，同样会降低查询性能。

**2、禁止给表中的每一列都建立单独的索引**

mysql5.6版本以后，虽然有了合并索引的优化方式，但是还是远远没有使用一个联合索引的查询方式好。

推荐：MySQL 索引B+树原理，以及建索引的几大原则。

**3、每个Innodb表必须有个主键**

Innodb是一种索引组织表：数据的存储的逻辑顺序和索引的顺序是相同的；表的存储顺序只能有一种 Innodb是按照主键索引的顺序来组织表的。

建议1：不要使用更新频繁的列作为主键，不适用多列主键（相当于联合索引） 

建议2：不要使用UUID、MD5、HASH、字符串列作为主键（无法保证数据的顺序增长）。

建议3：主键使用自增ID值。

**4、如何选择索引列的顺序**

区分度最高的放在联合索引的最左侧（区分度=列中不同值的数量/列的总行数）

尽量把字段长度小的列放在联合索引的最左侧（因为字段长度越小，一页能存储的数据量越大，IO性能也就越好）；

使用最频繁的列放到联合索引的左侧（这样可以比较少的建立一些索引）。

避免建立冗余索引和重复索引；因为这样会增加查询优化器生成执行计划的时间。

**5、优先考虑覆盖索引**

对于频繁的查询优先考虑使用覆盖索引；好处在于避免进行索引的二次查询，不必通过二级索引查到主键之后再去查询数据，减少了IO操作，提升了查询效率。

覆盖索引特点如下：

> 1、索引包含了(或覆盖)(或覆盖)查询语句中字段与条件的数据，即只需扫描索引而无须回表，包含了所有查询字段(select,where,join，ordery by,group by包含的字段
>
> 2、只能使用B-Tree索引做覆盖索引
>
> 3、EXPLAIN的Extra列可以看到“Using index”的信息

**6、避免使用外键约束**

外键会影响父表和子表的写操作从而降低性能。可在表与表之间的关联键上建立索引代替

**BTree 索引和Hash 索引的区别**

hash 索引相关特性如下：

* 只能使用 = 或 <=> 等操作符来表示比较
* 优化器不能使用Hash 索引来加速order by 操作
* MySQL 不能确定在两个值之间大约有多少行
* 只能使用整个关键字来搜索一行

而BTree 索引，当使用>、<、>=、<=、!=、<>或between、或like(后缀模糊) 操作符时，都可以使用相关列的索引



# 资料参考

* 《高性能 MySQL》
* https://www.toutiao.com/i6734868794180649485/