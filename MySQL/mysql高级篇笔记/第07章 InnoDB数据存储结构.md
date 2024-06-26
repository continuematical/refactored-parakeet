## 第07章 InnoDB数据存储结构

### 1. 数据库的存储结构：页

索引结构给我们提供了高效的索引方式，不过索引信息以及数据记录都保存在文件上的，确切说是存储在页结构中。另一方面，索引是在存储引擎中实现的，MySQL服务器上的`存储引擎`负责对表中数据的读取和写入工作。不同存储引擎中`存放的格式`一般不同的，甚至有的存储引擎比如Memory都不用磁盘来存储数据。

由于`InnoDB`是MySQL的`默认存储引擎`，所以本章剖析InooDB存储引擎的数据存储结构。

#### 1.1 磁盘与内存交互基本单位：页

InnoDB将数据划分为若干个页，InnoDB中页的大小默认为`16KB`（可以调整）。一般情况下会预留**1/16**的空间用于更新数据，真正使用的是**15/16**。一个页最少有***2***条数据，虚拟最小记录和最大记录。

以`页`作为磁盘和内存之间交互的基本单位，也就是一次最少从磁盘中读取16KB的内容到内存中，一次最少把内存中的16KB内容刷新到磁盘中。也就是说，**在数据库中，不论读一行，还是读多行，都是将这些行所在的页进行加载。也就是说，数据库管理存储空间的基本单位是页（Page），数据库I/O操作的最小单位是页。**一个页中可以存储多个行记录。

> 记录是按照行来存储的，但是数据库的读取并不以行为单位，否则一次读取（也就是一次I/O操作）只能处理一行数据，效率会非常低。

#### 1.2 页结构概述

页a、页b、页c...页n这些页可以`不在物理结构上相连`，只要通过`双向链表`相关联即可。每个数据页中的记录会按照主键值从小到大的顺序组成一个`单向链表`，每个数据页都会为存储在它里边的记录生成一个`页目录`，在通过主键查找某条记录的时候可以在页目录中`使用二分法`快速定位到对应的槽，然后再遍历该槽对应的分组中的记录即可快速找到指定的记录。

#### 1.3页的大小

不同的数据库管理系统页的大小不同，比如在MySQL中的InnoDB中，默认页的大小是`16KB` ，我们可以通过下面的命令来查看：

```mysql
mysql> show variables like '%innodb_page_size%';
```

![](D:\mysql-learning-master\mysql-learning-master\尚硅谷视频老师笔记\mysql高级篇笔记\images\InnoDB_Page_size.png)

`SQL Server` 中数据页的大小为`8KB` ，在`Oracle` 中用块表示页，它支持`2/4/8/16/32/64` KB。

#### 1.4 页的上层结构

![](D:\mysql-learning-master\mysql-learning-master\尚硅谷视频老师笔记\mysql高级篇笔记\images\InnoDB逻辑存储结构.png)

区（Extent）是比页大一级的存储结构，在InnoDB存储引擎中，一个区会分配`64个连续的页`。因为InnoDB中的页大小默认是16KB，所以一个区的大小是64*16KB=`1MB`。

段（Segment）由一个或多个区组成，区在文件系统是一个连续分配的空间（在InnoDB中是连续的64个页），不过在段中不要求区与区之间是相邻的。`段是数据库中的分配单位，不同类型的数据库对象以不同的段形式存在。`当我们创建数据表、索引的时候，就会相应创建对应的段，比如创建一张表时会创建一个表段，创建一个索引时会创建一个索引段。

表空间（Tablespace）是一个逻辑容器，表空间存储的对象是段，在一个表空间中可以有一个或多个段，但是一个段只能属于一个表空间。数据库由一个或多个表空间组成，表空间从管理上可以划分为`系统表空间`、`用户表空间`、`撤销表空间`、`临时表空间`等。

### 2. 页的内部结构

![img](https://gitee.com/MingZii/learn_db/raw/master/mysql/learn_mysql_ksf/%E7%AC%AC07%E7%AB%A0_InnoDB%E6%95%B0%E6%8D%AE%E5%AD%98%E5%82%A8%E7%BB%93%E6%9E%84.assets/image-20220324200934245.png)
$$
页(按类型划分)
\begin{cases}
数据页(保存B+树节点)\\
系统页\\
Undo页\\
事务数据页\\
……
\end{cases}
$$

$$
页(结构16KB)
\begin{cases}
文件头(38)\\
页头(56)\\
最大最小记录(26)\\
用户记录\\
空闲空间\\
页目录\\
文件尾(8)\\
\end{cases}
$$

![](D:\mysql-learning-master\mysql-learning-master\尚硅谷视频老师笔记\mysql高级篇笔记\images\InnoDB页结构.png)

##### 2.1 第1部分：文件头部和文件尾部

###### 2.1.1	File Header（文件头部）（38字节）

**作用**：
描述各种页的通用信息（比如页的编号、其上一页、下一页是谁等）。

**大小**：38字节

| 名称                               | 占用空间大小 | 描述                                                         |
| ---------------------------------- | ------------ | ------------------------------------------------------------ |
| `FIL_PAGE_SPACE_OR_CHKSUM`         | `4`字节      | 页的校验和（checksum值）                                     |
| `FIL_PAGE_OFFSET`                  | `4`字节      | 页号，唯一定位一个页                                         |
| `FIL_PAGE_PREV`                    | `4`字节      | 上一个页的页号                                               |
| `FIL_PAGE_NEXT`                    | `4`字节      | 下一个页的页号                                               |
| `FIL_PAGE_LSN`                     | `8`字节      | 日志序列号，页面被最后修改时对应的日志序列位置               |
| `FIL_PAGE_TYPE`                    | `2`字节      | 该页的类型                                                   |
| `FIL_PAGE_FILE_FLUSH_LSN`          | `8`字节      | 仅在系统表空间的一个页中定义，代表文件至少被刷新到了对应的LSN值 |
| `FIL_PAGE_ARCH_LOG_NO_OR_SPACE_ID` | `4`字节      | 页属于哪个表空间                                             |

- `FIL_PAGE_OFFSET（4字节）`：每一个页都有一个单独的页号，就跟你的身份证号码一样，InnoDB通过页号可以唯一定位一个页。
- `FIL_PAGE_TYPE（2字节）`：这个代表当前页的类型。
- `FIL_PAGE_SPACE_OR_CHKSUM` （文件头部和文件尾部都有）：对于一个很长的字符串来说，我们会通过某种算法来计算一个比较短的值来代表这个很长的字符串，这个比较短的值被称为`校验和` 。

作用：
InnoDB存储引擎以页为单位把数据加载到内存中处理，如果该页中的数据在内存中被修改了，那么在修改后的某个时间需要把数据同步到磁盘中。但是在同步了一半的时候断电了，造成了该页传输的不完整。
为了检测一个页是否完整(也就是在同步的时候有没有发生只同步一半的尴尬情况)，这时可以通过文件尾的校验和(`checksum` 值)与文件头的校验和做比对，如果两个值不相等则证明页的传输有问题，需要重新进行传输，否则认为页的传输已经完成。
具体的：
每当一个页面在内存中修改了，在同步之前就要把它的校验和算出来，因为`FileHeader` 在页面的前边，所以校验和会被首先同步到磁盘，当完全写完时，校验和也会被写到页的尾部，如果完全同步成功，则页的首部和尾部的校验和应该是一致的如果写了一半儿断电了，那么在`File Header` 中的校验和就代表着已经修改过的页，而在`File Trailer` 中的校验和代表着原先的页，二者不同则意味着同步中间出了错。这里校验方式就是采用Hash算法进行校验。

| 类型名称                | 十六进制 | 描述                             |
| ----------------------- | -------- | -------------------------------- |
| FIL_PAGE_TYPE_ALLOCATED | 0x0000   | 最新分配，还没有使用             |
| `FIL_PAGE_UNDO_LOG`     | 0x0002   | Undo日志页                       |
| FIL_PAGE_INODE          | 0x0003   | 段信息节点                       |
| FIL_PAGE_IBUF_FREE_LIST | 0x0004   | Insert Buffer空闲列表            |
| FIL_PAGE_IBUF_BITMAP    | 0x0005   | Insert Buffer位图                |
| `FIL_PAGE_TYPE_SYS`     | 0x0006   | 系统页                           |
| FIL_PAGE_TYPE_TRX_SYS   | 0x0007   | 事务系统数据                     |
| FIL_PAGE_TYPE_FSP_HDR   | 0x0008   | 表空间头部信息                   |
| FIL_PAGE_TYPE_XDES      | 0x0009   | 扩展描述页                       |
| FIL_PAGE_TYPE_BLOB      | 0x000A   | 溢出页                           |
| `FIL_PAGE_INDEX`        | 0x45BF   | 索引页，也就是我们所说的`数据页` |

- `FIL_PAGE_PREV（4字节）和FIL_PAGE_NEXT（4字节）`：InnoDB都是以页为单位存放数据的，如果数据分散到多个不连续的页中存储的话需要把这些页关联起来，FIL_PAGE_PREV和FIL_PAGE_NEXT就分别代表本页的上一个和下一个页的页号。这样通过建立一个双向链表把许许多多的页就都串联起来了，保证这些页之间**不需要是物理上的连续，而是逻辑上的连续。**
- `FIL_PAGE_SPACE_OR_CHKSUM（4字节）`：代表当前页面的校验和（checksum）。文件头部和文件尾部都有属性：FIL_PAGE_SPACE_OR_CHKSUM

**作用：**

InnoDB存储引擎以页为单位把数据加载到内存中处理，如果该页中的数据在内存中被修改了，那么`在修改后的某个时间需要把数据同步到磁盘中。`但是在同步了一半的时候断电了，造成了该页传输的不完整。

为了检测一个页是否完整（也就是在同步的时候有没有发生只同步一半的尴尬情况），这时可以通过文件尾的校验和（checksum 值）与文件头的校验和做比对，如果两个值不相等则证明页的传输有问题，需要重新进行传输，否则认为页的传输已经完成。

- `FIL_PAGE_LSN（8字节）`：页面被最后修改时对应的日志序列位置（英文名是：Log Sequence Number）

###### 2.1.2  File Trailer（文件尾部）（8字节）

- 前4个字节代表页的校验和：这个部分是和File Header中的校验和相对应的。
- 后4个字节代表页面被最后修改时对应的日志序列位置（LSN）：这个部分也是为了校验页的完整性的，如果首部和尾部的LSN值校验不成功的话，就说明同步过程出现了问题。

##### 2.2 第2部分：空闲空间、用户记录和最小最大记录

###### 2.2.1 Free Space (空闲空间)

我们自己存储的记录会按照指定的`行格式`存储到`User Records`部分。但是在一开始生成页的时候，其实并没有User Records这个部分，`每当我们插入一条记录，都会从Free Space部分，也就是尚未使用的存储空间中申请一个记录大小的空间划分到User Records部分`，当Free Space部分的空间全部被User Records部分替代掉之后，也就意味着这个页使用完了，如果还有新的记录插入的话，就需要去`申请新的页`了。

![](D:\mysql-learning-master\mysql-learning-master\尚硅谷视频老师笔记\mysql高级篇笔记\images\free_space.png)

###### 2.2.2 User Records (用户记录)

User Records中的这些记录按照`指定的行格式`一条一条摆在User Records部分，相互之间形成`单链表`。

###### 2.2.3 Infimum + Supremum（最小最大记录）

**记录可以比较大小吗**？
是的，记录可以比大小，对于一条完整的记录来说，比较记录的大小就是`比较主键`的大小。比方说我们插入的4行记录的主键值分别是：1、2、3、4，这也就意味着这4条记录是从小到大依次递增。

InnoDB规定的最小记录与最大记录这两条记录的构造十分简单，都是由5字节大小的记录头信息和8字节大小的一个固定的部分组成的。

![](D:\mysql-learning-master\mysql-learning-master\尚硅谷视频老师笔记\mysql高级篇笔记\images\最小最大记录.png)

这两条记录`不是我们自己定义的记录`，所以它们并不存放在页的User Records部分，他们被单独放在一个称为Infimum + Supremum的部分

![](D:\mysql-learning-master\mysql-learning-master\尚硅谷视频老师笔记\mysql高级篇笔记\images\最小最大记录具体.png)

##### 2.3 第3部分：页目录和页面头部

###### 2.3.1 Page Directory（页目录）

**为什么需要页目录**？
在页中，记录是以`单向链表`的形式进行存储的。单向链表的特点就是插入、删除非常方便，但是`检索效率不高`，最差的情况下需要遍历链表上的所有节点才能完成检索。因此在页结构中专门设计了页目录这个模块，`专门给记录做一个目录`，通过`二分查找法`的方式进行检索，提升效率。

**页目录，二分法查找**

1. 将所有的记录`分成几个组`，这些记录包括最小记录和最大记录，但*不包括*标记为“已删除”的记录。
2. 第 1 组，也就是最小记录所在的分组只有 ***1*** 个记录；
   最后一组，就是最大记录所在的分组，会有 ***1-8*** 条记录；
   其余的组记录数量在 ***4-8*** 条之间。
   这样做的好处是，除了第 ***1*** 组（最小记录所在组）以外，其余组的记录数会`尽量平分`。
3. 在每个组中最后一条记录的头信息中会存储该组一共有多少条记录，作为***n_owned*** 字段。
4. `页目录用来存储每组最后一条记录的地址偏移量`，这些地址偏移量会按照`先后顺序存储`起来，每组的地址偏移量也被称之为`槽（slot）`，每个槽相当于指针指向了不同组的最后一个记录。

**查找举例**

`select * from page_demo where c1 = 3` 

1. 方式一：顺序查找

从Infimum记录（最小记录）开始沿着链表一直查找，总会找到，因为链表中的节点是从小到大排序的，当链表某个节点大于你想要查找的主键值时，你就可以停止查找了。

如果一个页中存储了很多记录，那么性能会非常差。

2. 方式二：使用页目录二分查找

**举例：**

现在的page_demo表中正常的记录共有6条，InnoDB会把它们分成两组，第一组中只有一个最小记录，第二组中是剩余的5条记录。如下图：

![1680937022397](C:\Users\周锦\AppData\Roaming\Typora\typora-user-images\1680937022397.png)

- 现在页目录部分中有两个槽，也就意味着我们的记录被分成了两个组，槽1中的值是112，代表最大记录的地址偏移量（就是从页面的0字节开始数，数112个字节）；槽0中的值是99，代表最小记录的地址偏移量。
- 注意最小和最大记录的头信息中的n_owned属性
  - 最小记录的n_owned值为1，这就代表着以最小记录结尾的这个分组中只有1条记录，也就是最小记录本身。
  - 最大记录的n_owned值为5，这就代表着以最大记录结尾的这个分组中只有5条记录，包括最大记录本身还有我们自己插入的4条记录。

用箭头指向的方式替代数字，这样更易于我们理解，修改后如下

**为什么最小记录的n_owned值为1，而最大记录的n_owned值为5呢？**

InnoDB规定：对于最小记录所在的分组只能有1条记录，最大记录所在的分组拥有的记录条数只能在1~8条之间，剩下的分组中记录的条数范围只能在是 4~8 条之间。

分组是按照下边的步骤进行的：

- 初始情况下一个数据页里只有最小记录和最大记录两条记录，它们分属于两个分组。
- 之后每插入一条记录，都会从页目录中找到主键值比本记录的主键值大并且差值最小的槽，然后把该槽对应的记录的n_owned值加1，表示本组内又添加了一条记录，直到该组中的记录数等于8个。
- 在一个组中的记录数等于8个后再插入一条记录时，会将组中的记录拆分成两个组，一个组中4条记录，另一个5条记录。这个过程会在页目录中新增一个槽来记录这个新增分组中最大的那条记录的偏移量。

**页目录结构下如何快速查找信息**

![1680938272988](C:\Users\周锦\AppData\Roaming\Typora\typora-user-images\1680938272988.png)

如果我们想查找主键值为6的记录，初始情况下最低槽的位置是low=0，最高为high=4；

1. 计算中间槽的位置（0+4）/2=2，查看槽2的值为8，high=2，low=0；
2. （0+2）/2=1，槽1的值为4，low=1，high=2；
3. 沿着单链表遍历槽2的记录。

**如何定位一个槽中最小的记录**

每个槽在物理内存上是挨着的，如果获取槽2的最小记录，只需拿到槽1的最小记录，该条记录的下一条记录就是槽2的最小记录。

###### 2.3.2 Page Header（页面头部）

为了能得到一个数据页中存储的记录的状态信息，比如本页中已经存储了多少条记录，第一条记录的地址是什么，页目录中存储了多少个槽等等，特意在页中定义了一个叫***Page Header***的部分，这个部分占用固定的***56***个字节，专门存储各种状态信息。

| 名称              | 占用空间大小 | 描述                                                         |
| ----------------- | ------------ | ------------------------------------------------------------ |
| PAGE_N_DIR_SLOTS  | 2字节        | 在页目录中的槽数量                                           |
| PAGE_HEAP_TOP     | 2字节        | 还未使用的空间最小地址，也就是说从该地址之后就是`Free Space` |
| PAGE_N_HEAP       | 2字节        | 本页中的记录的数量（包括最小和最大记录以及标记为删除的记录） |
| PAGE_FREE         | 2字节        | 第一个已经标记为删除的记录的记录地址（各个已删除的记录通过`next_record`也会组成一个单链表，这个单链表中的记录可以被重新利用） |
| PAGE_GARBAGE      | 2字节        | 已删除记录占用的字节数                                       |
| PAGE_LAST_INSERT  | 2字节        | 最后插入记录的位置                                           |
| PAGE_DIRECTION    | 2字节        | 记录插入的方向                                               |
| PAGE_N_DIRECTION  | 2字节        | 一个方向连续插入的记录数量                                   |
| PAGE_N_RECS       | 2字节        | 该页中记录的数量（不包括最小和最大记录以及被标记为删除的记录） |
| PAGE_MAX_TRX_ID   | 8字节        | 修改当前页的最大事务ID，该值仅在二级索引中定义               |
| PAGE_LEVEL        | 2字节        | 当前页在B+树中所处的层级                                     |
| PAGE_INDEX_ID     | 8字节        | 索引ID，表示当前页属于哪个索引                               |
| PAGE_BTR_SEG_LEAF | 10字节       | B+树叶子段的头部信息，仅在B+树的Root页定义                   |
| PAGE_BTR_SEG_TOP  | 10字节       | B+树非叶子段的头部信息，仅在B+树的Root页定义                 |

PAGE_DIRECTION
假如新插入的一条记录比上一条记录的主键值大，我们就说这条记录的插入方向是***右边***，否则是***左边***。用来表示最后一条记录的插入方向。

PAGE_N_DIRECTION
连续***n***次插入数据的方向都是一致的，InnoDB会把沿着同一个方向插入数据的***条数***记录下来，当然，如果最后一条数据的插入方向发生了改变，则这个状态的值就会重新清零统计。

###### 2.3.3 从数据页的角度看B+树如何查询

1. B+树是如何查询记录的

如果通过B+树的索引查询行记录，首先是从B+树的根开始，逐层检索，直到找到叶子节点，也就是找到对应的数据页为止。将数据页加载到内存，页目录中的槽采用二分查找的方式先找到一个粗略的分组记录，然后在记录里面通过链表遍历的方式查找。

2. *普通索引*和*唯一索引*

唯一索引：索引上添加约束性，关键字唯一；
我们读取一条记录是将该记录所在的页面都加载到内存中进行读取，如果使用普通索引的字段上进行查找也就是在内存中多判断几下的操作，对于CPU来说可以忽略不计。

### 3. InnoDB行格式（或记录格式）

$$
InnoDB行格式
\begin{cases}
Antelope
\begin{cases}
compact\\
redundant
\end{cases}
\\
Barracuda
\begin{cases}
compressed\\
dynamic\\
\end{cases}
\\
\end{cases}
$$

#### 3.1 指定行格式的语法

```mysql
CREATE TABLE 表名 (列的信息) ROW_FORMAT=行格式名称
```

```mysql
ALTER TABLE 表名 ROW_FORMAT=行格式名称
```

查看行格式

```mysql
show variables like "row_format";
```

#### 3.2 COMPACT行格式

在MySQL 5.1版本中，默认设置为***Compact***行格式。一条完整的记录其实可以被分为记录的额外信息和记录的真实数据两大部分。

##### 3.2.1 变长字段长度列表

MySQL支持一些变长的数据类型，比如VARCHAR(M)、VARBINARY(M)、TEXT类型，BLOB类型，这些数据类型修饰列称为`变长字段`，变长字段中存储多少字节的数据不是固定的，所以我们在存储真实数据的时候需要顺便把这些数据占用的字节数也存起来。`在Compact行格式中，把所有变长字段的真实数据占用的字节长度都存放在记录的开头部位，从而形成一个变长字段长度列表。`

> 注意：这里面存储的变长长度和字段顺序是反过来的。比如两个varchar字段在表结构的顺序是a(10)，b(15)。那么在变长字段长度列表中存储的长度顺序就是15，10，是反过来的。

##### 3.2.2 NULL值列表

Compact行格式会把可以为NULL的列统一管理起来，存在一个标记为NULL值列表中。如果表中没有允许存储 NULL 的列，则 NULL值列表也不存在了。
**为什么定义NULL值列表？**
之所以要存储NULL是因为数据都是需要***`对齐`***的，如果`没有标注出来NULL值的位置`，就有可能在查询数据的时候`出现混乱`。如果使用`一个特定的符号`放到相应的数据位表示空置的话，虽然能达到效果，但是这样很浪费空间，所以直接就在行数据得头部开辟出一块空间专门用来记录该行数据哪些是非空数据，哪些是空数据，格式如下：

1. 二进制位的值为1时，代表该列的值为NULL。
2. 二进制位的值为0时，代表该列的值不为NULL。

> 注意：同样顺序也是反过来存放的

##### 3.2.3 记录头信息（5字节）

| 名称            | 大小（单位：bit） | 描述                                                         |
| --------------- | ----------------- | ------------------------------------------------------------ |
| `预留位1`       | 1                 | 没有使用                                                     |
| `预留位2`       | 1                 | 没有使用                                                     |
| `delete_mask`   | 1                 | 标记该记录是否被删除                                         |
| `mini_rec_mask` | 1                 | B+树的每层非叶子节点中的最小记录都会添加该标记               |
| `n_owned`       | 4                 | 表示当前记录拥有的记录数                                     |
| `heap_no`       | 13                | 表示当前记录在记录堆的位置信息                               |
| `record_type`   | 3                 | 表示当前记录的类型，`0`表示普通记录，`1`表示B+树非叶子节点记录，`2`表示最小记录，`3`表示最大记录 |
| `next_record`   | 16                | 表示下一条记录的相对位置                                     |

- `delete_mask`：这个属性标记着当前记录是否被删除，占用1个二进制位。
  - 值为0：代表记录并没有被删除
  - 值为1：代表记录被删除掉了

**被删除的记录为什么还在页中存储呢？**
你以为它删除了，可它还在真实的磁盘上。这些被删除的记录之所以不立即从磁盘上移除，是因为移除它们之后其他的记录在磁盘上需要`重新排列，导致性能消耗`。所以只是打一个删除标记而已，所有被删除掉的记录都会组成一个所谓的`垃圾链表`，在这个链表中的记录占用的空间称之为`可重用空间`，之后如果有新记录插入到表中的话，可能把这些被删除的记录占用的存储空间覆盖掉。

- `min_rec_mask`：B+树的每层非叶子节点中的最小记录都会添加该标记，min_rec_mask值为1。我们自己插入的四条记录的min_rec_mask值都是0，意味着它们都不是B+树的非叶子节点中的最小记录。
- `record_type`：这个属性表示当前记录的类型，一共有4种类型的记录：
  - 0：表示普通记录
  - 1：表示B+树非叶节点记录
  - 2：表示最小记录
  - 3：表示最大记录
- `heap_no`：这个属性表示当前记录在本页中的位置。

**怎么不见heap_no值为0和1的记录呢**？
MySQL会自动给每个页里加了两个记录，由于这两个记录并不是我们自己插入的，所以有时候也称为`伪记录`或者`虚拟记录`。这两个伪记录一个代表`最小记录`，一个代表`最大记录`。最小记录和最大记录的heap_no值分别是0和1，也就是说它们的位置最靠前。

- `n_owned`：页目录中每个组中最后一条记录的头信息中会存储该组一共有多少条记录，作为`n_owned` 字段。
- `next_record`：记录头信息里该属性非常重要，它表示从当前记录的真实数据到下一条记录的真实数据的`地址偏移量`。

比如说：第一条语句的next_record为32，意味着第一条数据真实地址后的32个字节后是下一条数据。

注意：并不是插入的下一条顺序，而是按照主键顺序由小到大的下一条记录。而且规定Infimum (最小记录)的下一条记录是本页主键值最小的记录，本页主键值最大的下一条记录也就是Supremum记录。

![1680884675011](C:\Users\周锦\AppData\Roaming\Typora\typora-user-images\1680884675011.png)

![1680884838795](C:\Users\周锦\AppData\Roaming\Typora\typora-user-images\1680884838795.png)

当数据页存在多条被删除的记录时，这些记录的next_record属性将会把删除掉的记录组成一个垃圾链表，以备之后重用这部分储存空间。

##### 3.2.4 记录的真实数据

| 列名           | 是否必须 | 占用空间 | 描述                   |
| -------------- | -------- | -------- | ---------------------- |
| row_id         | 否       | 6字节    | 行ID，唯一标识一条记录 |
| transaction_id | 是       | 6字节    | 事务ID                 |
| roll_pointer   | 是       | 7字节    | 回滚指针               |

实际上这几个列的真正名称是DB_ROW_ID,DB_TRS_ID,DB_ROLL_PTR；

一个表没有手动定义主键，则会选取一个Unique键作为主键，如果连Unique键都没有定义的话，则会为表默认添加一个名为row_id的隐藏列作为主键。所以row_id是在**没有自定义主键以及Unique键的情况下**才会存在的。

#### 3.3 Dynamic和Compressed行格式

##### 3.3.1   案例

如果我们使用ASCII码字符集，一个字符表示一个字节，此时VARCHAR(65535)实际不可用：
`create table manager(name varchar(65535)) charset=ascii row_format=compact;` 

实际超出最大存储空间，因为65535个字节不仅代表实际数据长度，还包括储存的额外信息；
`create table manager(name varchar(65532)) charset=ascii row_format=compact;` 
数据长度（65532）+变长字段长度列表（2）+NULL值列表（1）；
`create table manager(name varchar(65533)) not null charset=ascii row_format=compact;` 
数据长度（65532）+变长字段长度列表（2）；

##### 3.3.2  行溢出

InnoDB存储引擎可以将一条记录的某些数据存储在真正的数据页面之外；

我们可以知道一个页的大小一般是16KB，也就是16384字节，而一个VARCHAR(M)类型的列就最多可以存储65533个字节，这样就可能出现一个页存放不了一条记录，这种现象称为`行溢出`

在**Compact**和**Reduntant**行格式中，对于占用存储空间非常大的列，在记录的真实数据处只会存储该列的一部分数据，把剩余的数据分散存储在几个其他的页中进行`分页存储`，然后记录的真实数据处用***20***个字节存储指向这些页的地址（当然这20个字节中还包括这些分散在其他页面中的数据的占用的字节数），从而可以找到剩余数据所在的页。这称为`页的扩展`。

在MySQL 8.0中，默认行格式就是Dynamic，Dynamic、Compressed行格式和Compact行格式挺像，只不过在处理行溢出数据时有分歧：

- Compressed和Dynamic两种记录格式对于存放在BLOB中的数据采用了完全的行溢出的方式。在数据页中只存放20个字节的指针（溢出页的地址），实际的数据都存放在Off Page（溢出页）中。
- Compact和Redundant两种格式会在记录的真实数据处存储一部分数据（存放768个前缀字节）。

**Compressed**行记录格式的另一个功能是存储在其中的行记录会以***zlib***的算法进行压缩，因此对于***BLOB***，***TEXT***，***VARCHAR***这种大长度类型的数据能够进行非常有效的存储。

#### 3.4 Redundant行格式

MySQL5.0之前InnoDB记录数据的方式，现在仍然支持Redundant是为了以前版本的兼容；

![1680965784222](C:\Users\周锦\AppData\Roaming\Typora\typora-user-images\1680965784222.png)

1. 字段长度偏移列表

所有列的长度信息逆序存放在列表中，根据两个相邻数值的差值来计算各个长度。
所有字段的长度都表示了，NULL值列表也就没有必要了（相差为0）。

2. 记录头信息（***6***个字节）

### 4. 区、段和碎片区

![img](https://gitee.com/MingZii/learn_db/raw/master/mysql/learn_mysql_ksf/%E7%AC%AC07%E7%AB%A0_InnoDB%E6%95%B0%E6%8D%AE%E5%AD%98%E5%82%A8%E7%BB%93%E6%9E%84.assets/image-20220324200502569.png)

#### 4.1 为什么要有区？

`B+`树的每一层中的页都会形成一个双向链表，如果是以`页为单位`来分配存储空间的话，双向链表相邻的两个页之间的`物理位置可能离得非常远`。我们介绍B+树索引的使用场景的时候特别提到范围查询只需要定位到最左边的记录和最右边的记录，然后沿着双向链表一直扫描就可以了，而如果链表中相邻的两个页物理位置离得非常远，就是所谓的`随机I/O`。再一次强调，磁盘的速度和内存的速度差了好几个数量级，`随机I/O是非常慢`的，所以我们应该**尽量**让链表中相邻的页的物理位置也相邻，这样进行范围查询的时候才可以使用所谓的`顺序I/O`。

引入`区`的概念，一个区就是物理位置上连续的***64***`个页`。因为***InnoDB***中的页的大小默认是16KB，所以一个区的大小是**64*16KB**=`1MB`。在表中`数据量大`的时候，为某个索引分配空间的时候就不再按照页的单位分配了，而是按照`区为单位分配`，甚至在表中的数据特别多的时候，可以一次性分配多个连续的区。虽然可能造成`一点点空间的浪费`（数据不足以填充满整个区），但是从性能角度看，可以消除很多的随机I/O，`功大于过`！

#### 4.2 为什么要有段？

对于范围查询，其实是对**B+**树叶子节点中的记录进行顺序扫描，而如果不区分叶子节点和非叶子节点，统统把节点代表的页面放到申请到的区中的话，进行范围扫描的效果就大打折扣了。所以**InnoDB**对**B+**树的`叶子节点`和`非叶子节点`进行了区别对待，也就是说叶子节点有自己独有的区，非叶子节点也有自己独有的区。存放叶子节点的区的集合就算是一个`段（segment）`，存放非叶子节点的区的集合也算是一个段。也就是说一个索引会生成2个段，一个`叶子节点段`，一个`非叶子节点段`。

除了索引的叶子节点段和非叶子节点段之外，InnoDB中还有为存储一些特殊的数据而定义的段，比如回滚段。所以，常见的段有<u>数据段`、`索引段`、`回滚段</u>。数据段即为B+树的叶子节点，索引段即为B+树的非叶子节点。
$$
段
\begin{cases}
叶子节点段\quad叶子节点区的集合\\
非叶子节点段\quad非叶子节点区的集合\\
数据段\\
索引段\\
回滚段\\
...
\end{cases}
$$


在***InnoDB***存储引擎中，对段的管理都是由引擎自身所完成，***DBA***不能也没有必要对其进行控制。这从一定程度上简化了DBA对于段的管理。

段其实不对应表空间中的某一个连续的物理区域，而是一个逻辑上的概念，由若干个零散的页面以及一些完整的区组成。

#### 4.3 为什么要有碎片区？

默认情况下，一个使用InnoDB存储引擎的表只有一个聚簇索引，一个索引会生成2个段，而段是以区为单位申请存储空间的，一个区默认占用1M（***64\*16KB=1024KB***）存储空间，所以**默认情况下一个只存在几条记录的小表也需要2M的存储空间么？**以后每次添加一个索引都要多申请2M的存储空间么？这对于存储记录比较少的表简直是天大的浪费。这个问题的症结在于到现在为止我们介绍的区都是非常`纯粹`的，也就是一个区被整个分配给某一个段，或者说区中的所有页面都是为了存储同一个段的数据而存在的，即使段的数据填不满区中所有的页面，那余下的页面也不能挪作他用。

为了考虑以完整的区为单位分配给某个段对于`数据量较小`的表太浪费存储空间的这种情况，InnoDB提出了一个`碎片（fragment）区`的概念。在一个碎片区中，并不是所有的页都是为了存储同一个段的数据而存在的，而是碎片区中的页可以用于不同的目的，比如有些页面用于段A，有些页面用于段B，有些页甚至哪个段都不属于。`碎片区直属于表空间`，并不属于任何一个段。

所以此后为某个段分配存储空间的策略是这样的：

- 在刚开始向表中插入数据的时候，段是从某个碎片区以单个页面为单位来分配存储空间的。
- 当某个段已经占用了`32个碎片区`页面之后，就会申请以完整的区为单位来分配存储空间。

所以现在段不能仅定义为是某些区的集合，更精确的应该是`某些零散的页面`已经`一些完整的区`的集合。

#### 4.4 区的分类

区大体上可以分为4种类型：

- `空闲的区(FREE)`：现在还没有用到这个区中的任何页面。
- `有剩余空间的碎片区(FREE_FRAG)`：表示碎片区中还有可用的页面。
- `没有剩余空间的碎片区(FULL_FRAG)`：表示碎片区中的所有页面都被使用，没有空闲页面。
- `附属于某个段的区(FSEG)`：每一索引都可以分为叶子节点段和非叶子节点段

处于`FREE`、`FREE_FRAG`以及`FULL_FRAG`这三种状态的区都是独立的，直属于表空间。而处于`FSEG`状态的区是附属于某个段的。

### 5.	表空间

表空间可以看作是InnoDB存储引擎逻辑结构的最高层，所有的数据都存放在表空间，数据库初始化创建了一个ibdatal的表空间文件。

表空间是一个逻辑容器，表空间的存储对象是段，在一个表空间可以由一个或多个段，但是一个段只能属于一个表空间。表空间数据库是由一个或多个表空间组成，表空间从管理上可以划分为系统表空间，独立表空间，撤销表空间和临时表空间等。

#### 5.1	独立表空间

每个表都会有一个独立表空间，也就是**数据**和**索引信息**都会保存在自己的表空间中，其他信息还是存储在系统表空间。独立的表空间可以在不同的数据库之间进行迁移。

![](D:\mysql-learning-master\mysql-learning-master\尚硅谷视频老师笔记\mysql高级篇笔记\images\独立表空间和系统表空间.png)

空间可以回收（DROP TABLE操作可以自动回收表空间，其他情况表空间不能回收）。如果对于统计分析或者是日志表，删除大量数据可以通过：**ALTER TABLE tablename ENGINEE=InnoDB**或者**pt-online_schema_change**；回收不用的空间。对于使用独立表空间的表，不管怎么删除，表空间的碎片不会太严重地影响性能，反而有机会处理。

但是共享表空间不能回收，如果想要回收需要备份数据库、删除原表，然后将数据导入到和原表一样的数据结构中，才能删除。统计类分析、日志文件不适合共享表空间。

##### 5.1.1	独立表空间结构

段，区，页

##### 5.1.2	真实表空间对应的文件大小

数据目录下，新建的表对应的`.ibd` 文件只用了96KB（***MySQL5.7***），这是因为一开始表空间占用的内存很少，因为表里面没有数据。不过别忘了这些`.ibd` 文件是自扩展的，随着表中数据的增多，表空间对应的文件页逐渐增大。

##### 5.1.3	查看InnoDB的表空间类型

```mysql
show variables like 'innodb_file_per_table';
```

#### 5.2	系统表空间

系统表空间的结构和独立表空间的基本类似，只不过整个MySQL只有一个系统表空间。系统表空间会额外记录一些有关整个系统信息的页面，这部分是独立表空间所没有的。

##### 5.2.1 InnoDB数据字典

每当我们向一个表中插入一条记录的时候，MySQL校验过程如下：

先要校验一下插入语句对应的表存不存在，插入的列和表中的列是否符合，如果语法没有问题的话还需要知道该表的聚簇索引和所有二级索引对应的根页面是哪个表空间的页面，然后把记录插入到对应的B+树之间。所以说，MySQL除了保存我们插入的用户数据，还需要保存许多额外的信息，比如说：

> 某个表属于哪个表空间，表里面有多少列；
> 表对应的每一个列的类型是什么；
> 该表有多少索引，每个索引对应哪几个字段，该索引对应的根页面再哪一个表空间的哪一个页面；
> 该表有哪些外键，外键对应表中的哪些列；
> 某个表空间对应文件系统上文件路径是什么；
> ...

上述数据不是我们使用***INSERT***语句插入的用户数据，实际上是为了更好地管理用户数据而不得已引入的一些额外数据，这些数据也称为元数据。InnoDB存储引擎特地定义了一些列的内部系统表来记录这些元数据：

![1681023504800](C:\Users\周锦\AppData\Roaming\Typora\typora-user-images\1681023504800.png)

这些系统表也被称为数据字典，它们都是以B+树的形式保存到系统表空间的某些页面中，这四个表尤为重要，被称为基本系统表；

##### 5.2.2	基本系统表

###### ***SYS_TABLES***

![SYS_TABLE](C:\Users\周锦\AppData\Roaming\Typora\typora-user-images\1681023795064.png)

###### ***SYS_COLUMNS***

![1681023890553](C:\Users\周锦\AppData\Roaming\Typora\typora-user-images\1681023890553.png)

###### ***SYS_INDEXS***

![1681023910272](C:\Users\周锦\AppData\Roaming\Typora\typora-user-images\1681023910272.png)

###### ***SYS_FIELDS***

![1681023935977](C:\Users\周锦\AppData\Roaming\Typora\typora-user-images\1681023935977.png)

注意：用户不能直接访问系统表，除非直接去解析对应文件系统之间的文件，不过系统数据库innodb_scheme提供了一些以innodb_sys开头的表：

```mysql
show variables like '%innnodb_sys%';
```

##### 5.3	临时表空间

空间名为ibtmpl1，默认大小为**12MB**。

把临时表的数据从系统空间中抽取出来，形成自己的独立表空间参数innodb_temp_data_file_path。

### 6.	数据页加载的三种方式

InnoDB从磁盘中读取数据的最小单位是数据页，对于***MySQL***存放的数据，逻辑理念上我们称之为表，但在磁盘等物理层面上是按数据页的形式存放的，当其加载到***MySQL***中我们称之为缓存页。

如果缓冲池没有该页数据，那么缓冲池有以下三种读取数据的方法：

#### 6.1	内存读取

如果该数据位于内存中，基本执行时间是在1ms左右，效率很高；

内存->记录->数据库缓冲池。

![img](https://gitee.com/MingZii/learn_db/raw/master/mysql/learn_mysql_ksf/%E7%AC%AC07%E7%AB%A0_InnoDB%E6%95%B0%E6%8D%AE%E5%AD%98%E5%82%A8%E7%BB%93%E6%9E%84.assets/image-20220325104247259.png)

#### 6.2	随机读取

如果数据没有在内存中，则需要在磁盘上进行查找，整体时间预估在10ms左右，这10ms中有6ms是磁盘的实际繁忙时间（包括4ms寻遍和2ms半圈旋转时间），3ms是可能发生的排队时间的预估值，另外还有1ms的传输时间，将页从磁盘服务器缓冲区 传输到数据库服务缓冲区中，看起来很快，但是对数据库来说消耗的时间已经够长了，因为这只是一个页的读取时间。

磁盘->记录->数据库缓冲池。

![img](https://gitee.com/MingZii/learn_db/raw/master/mysql/learn_mysql_ksf/%E7%AC%AC07%E7%AB%A0_InnoDB%E6%95%B0%E6%8D%AE%E5%AD%98%E5%82%A8%E7%BB%93%E6%9E%84.assets/image-20220325104508228.png)

#### 6.3	顺序读取

顺序存储实际上是一种批量读取的方式，因为我们请求的数据往往在磁盘上都是相邻存储的，顺序存储可以帮助我们批量读取数据，这样的话，一次性加载到缓冲池就不需要再对其他页面进行磁盘I/O操作了，如果一个磁盘的吞吐量是40MB/s，那么对于一个16KB大小的页来说，一次性可以顺序读取2560个页，相当于一个页的读取时间是0.4ms，采用批量读取的方式，即使是从磁盘上进行读取，效率也比从内存中单独读取一个页的效率要高。

### 7.	内存结构

实际上MySQL内存的组成和[Oracle](https://so.csdn.net/so/search?q=Oracle&spm=1001.2101.3001.7020)类似，也可以分为SGA(系统全局区)和PGA(程序缓存区)。

```mysql
show variables like "%buffer%"
```

#### 7.1	SGA

1. innodb_buffer_bool
   用来缓存Innodb表的数据、索引、插入缓冲、数据字典等信息。

2. innodb_log_buffer
   事务在内存中的缓冲，即redo log buffer的大小

3. query cache
   高速查询缓存，在生产环境中建议关闭。

4. key_buffer_size
   用于MyISAM存储引擎，缓存MyISAM存储。引擎表的索引文件(区别于innodb_buffer_poll数据和索引缓存)

5. innodb_additional_mem_pool_size
   用来缓存数据字典信息和其它内部数据结构的内存池的大小，MySQL5.7.4中该参数取消。

#### 7.2 PGA

1. sort_buffer_size

主要用于SQL语句在内存中的临时排序

2. join_buffer_size

表连接使用，用于BKA，MySQL5.6之后开始支持。

3. read_buffer_size

表顺序扫描的缓存，只能应用于MyISAM表存储引擎。

4. read_rnd_buffer_size

MySQL随机读缓冲区大小，用于做mrr,mrr是MySQL5.6之后才有的特性。

5. tmp_table_size

SQL语句在排序或分组时没有用到索引，就会使用临时表空间。

6. max_heap_table_size

管理heap，memory存储引擎表。

### 8.	Buffer状态及其链表结构

page是InnoDB磁盘I/O的最小单位，数据存放在page中，对应到内存就是一个个buffer。

#### 8.1	状态

buffer有三种状态：

1. free buffer：这个状态下的buffer没有被使用，是空闲的。但是在生产环境中，数据库很繁忙的情况下，free buffer基本不存在；

2. clean buffer：内存中buffer里的数据和磁盘中page的数据一致；

3. dirty buffer：内存中新写入的数据还没有刷新到磁盘，buffer里的数据和磁盘中page的数据不一致；

#### 8.2	链表

buffer在内存中，需要被chain(链)组织起来，InnoDB是双向链表结构，由三种不同的buffer状态衍生出三条链表

1. free list：把free buffer都串联起来。数据库跑起来的时候，每次把page调到内存中，都会判断free buffer是否够用，不够用的话从lru list和flush list中释放free buffer；

2. lru list：把与磁盘数据一致，并且最近最少被使用的buffer串联起来，释放出free buffer；

3. flush list：把dirty buffer串联起来，方便刷新线程将脏数据刷到磁盘。规则是将那些最近最少被弄脏的数据串起来，刷新到磁盘后，释放出更多的free buffer。



