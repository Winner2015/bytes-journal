

# 1、MySQL的存储结构

## 1.1 MySQL的体系架构

ySQL 从第一个版本发布到现在已经有了 20 多年的历史，整个应用的体系结构变得越来越复杂，官方的架构图太庞杂，返璞归真，来看一个简化版的MySQL架构图：

![0](0.png)

连接池、查询缓存、sql解析器、sql优化器等不是MySQL独有的东西，在其他主流数据库中都有各自的组件，只是实现的方式不尽相同而已。

MySQL区别于其他数据库的最重要的一个特点就是其**插件式的表存储引擎**。存储引擎是底层物理结构的实现，能够根据具体的应用建立不同的存储引擎表，也就是说，**存储引擎是基于表的，而不是数据库**。

## 1.2 逻辑结构

在MySQL中，我们可以使用不同的存储引擎来存储数据，而绝大多数存储引擎都以二进制的形式存储数据。

在InnoDB存储引擎中，所有数据都被逻辑地存放在一个空间内，称为**表空间**（tablespace），而表空间由**段**（sengment）、**区**（extent）、**页**（page）组成。

![1](1.png)

### 1.2.1 表空间

表空间可以看做是InnoDB存储引擎逻辑结构的最高层，所有的数据都存放在表空间中。在默认情况下 InnoDB存储引擎有一个共享表空间 `ibdata1`，即所有数据都存放在这个表空间内。如果用户启用了参数 `innodb_file_per_table`，则每张表内的数据可以单独放到一个表空间内。

### 1.2.2 段

表空间是由各个段组成的，常见的段有数据段、索引段、回滚段等。InnoDB下的都是索引组织表，因此数据即索引，索引即数据。那么数据段即为B+树段叶子节点，索引段即为B+树段非索引节点。

### 1.2.3 区

区是由连续的页组成的空间，**在任何情况下每个区大小都为1MB**，为了保证页的连续性，InnoDB存储引擎每次从磁盘一次申请4-5个区。默认情况下，InnoDB的页大小默认为16KB，即一个区中有64个连续的页。

在建表的时候可以开启表压缩，能够使表中的数据以压缩格式存储，在一定情况下可以提高原生性能和可伸缩性。

在创建一个压缩表之前，需要启用独立表空间参数`innodb_file_per_table=1`；也需要设置`innodb_file_format=Barracuda`。

```mysql
SET GLOBAL innodb_file_per_table=1;
SET GLOBAL innodb_file_format=Barracuda;
CREATE TABLE t1
 (c1 INT PRIMARY KEY) 
 ROW_FORMAT=COMPRESSED  
 KEY_BLOCK_SIZE=8;
```

`KEY_BLOCK_SIZE`的值作为一种提示，如必要，Innodb也可以使用一个不同的值。KEY_BLOCK_SIZE的值必须小于等于`innodb page size`。0代表默认压缩页的值，为Innodb页的一半。

有一点需要说明，在用户启用了参数 `innodb_file_per_talbe`后，创建的表默认大小是96KB，而不是一个区的大小。这是因为在每个段开始时，先用32个页大小的碎片页( fragment page)来存放数据，在使用完这些页之后才是64个连续页的申请。这样做的目的是，对于一些小表，或者是undo这类的段，可以在开始时申请较少的空间，节省磁盘容量的开销。

### 1.2.4 页

页是InnoDB磁盘管理的最小单位，**每个页默认16KB**，通过参数`innodb_page_size`可以将默认页的大小设置为4K、8K等。

```mysql
mysql> show variables like 'innodb_page_size'\G;
*************************** 1. row ***************************
Variable_name: innodb_page_size
        Value: 16384
```

innoDB存储引擎中，常见的页类型有：

- 数据页（B-tree Node)
- undo页（undo Log Page）
- 系统页 （System Page）
- 事物数据页 （Transaction System Page）
- 插入缓冲位图页（Insert Buffer Bitmap）
- 插入缓冲空闲列表页（Insert Buffer Free List）
- 未压缩的二进制大对象页（Uncompressed BLOB Page）
- 压缩的二进制大对象页 （compressed BLOB Page）

### 1.2.5 行

InnoDB是按行进行存放的，每个页存放的行记录也是有硬性定义的，最多允许存放16KB/2-200，即7992行记录（据说是由内核定义的）。

## 1.3 物理结构

### 1.3.1 表文件

不论使用何种存储引擎，每个表都有一个以`.frm`为后缀的文件，记录了该表的结构定义。

对于InnoDB而言，有一个表空间（tableplace）的概念，用于存放各种数据。可以通过参数`innodb_data_file_path`配置：

```mysql
mysql> show variables like 'innodb_data_file_path'\G;
*************************** 1. row ***************************
Variable_name: innodb_data_file_path
        Value: ibdata1:12M:autoextend
```

可见，默认情况下，表空间的名字为`ibdata1`，初始大小为12M，可以自动增长。

ibdata1也被称为共享表空间，所有基于InnoDB的表数据都会记录到共享表空间。

如果想给每个表使用独立表空间，可以通过参数`innodb_file_per_table`开启：

```mysql
mysql> show variables like 'innodb_file_per_table'\G;
*************************** 1. row ***************************
Variable_name: innodb_file_per_table
        Value: ON
```

独立表空间的命名规则为：表名.ibd

需要注意的是，**独立表空间文件仅存储该表的数据、索引和插入缓冲等信息，其余信息（两次写缓冲等）还是存放于共享表空间**。

### 1.3.2 数据页结构

页是 InnoDB 存储引擎管理数据的最小磁盘单位，而 B-Tree 节点就是实际存放表中数据的页面。

一个 InnoDB 页有以下七个部分：

![2](2.png)

每一个页中包含了两对 header/trailer：内部的 Page Header/Page Directory 关心的是页的状态信息，而 Fil Header/Fil Trailer 关心的是记录页的头信息。

在页的头部和尾部之间就是用户记录和空闲空间了，每一个数据页中都包含 Infimum 和 Supremum 这两个**虚拟**的记录（可以理解为占位符），Infimum 记录是比该页中任何主键值都要小的值，Supremum 是该页中的最大值：

![3](3.png)

User Records 就是整个页面中真正用于存放行记录的部分，而 Free Space 就是空余空间了，它是一个链表的数据结构，为了保证插入和删除的效率，整个页面并不会按照主键顺序对所有记录进行排序，它会自动从左侧向右寻找空白节点进行插入，行记录在物理存储上并不是按照顺序的，它们之间的顺序是由 `next_record` 这一指针控制的。

B+ 树在查找对应的记录时，并不会直接从树中找出对应的行记录，它只能获取记录所在的页，将整个页加载到内存中，再通过 Page Directory 中存储的稀疏索引和 `n_owned`、`next_record` 属性取出对应的记录，不过因为这一操作是在内存中进行的，所以通常会忽略这部分查找的耗时。

### 1.3.3 行记录格式

当 InnoDB 存储数据时，可以使用不同的行格式进行存储，具体的存储格式可以通过命令`show table status`来查看：

```mysql
mysql> show table status like 'servers'\G;
*************************** 1. row ***************************
           Name: servers
         Engine: InnoDB
        Version: 10
     Row_format: Dynamic
           Rows: 0
 Avg_row_length: 0
    Data_length: 16384
Max_data_length: 0
   Index_length: 0
      Data_free: 0
 Auto_increment: NULL
    Create_time: 2018-12-23 20:23:07
    Update_time: NULL
     Check_time: NULL
      Collation: utf8_general_ci
       Checksum: NULL
 Create_options: stats_persistent=0
        Comment: MySQL Foreign Servers table
```

其中，`row_format`属性即代表行记录的结构类型。

MySQL 5.7 版本支持以下格式的行存储方式：

![4](4.png)

Compact和Redundant格式称为Antelope文件格式，Compact 和 Redundant 在磁盘上按照以下方式存储：

![5](5.png)

Compact 和 Redundant 格式最大的不同就是记录格式的第一个部分；在 Compact 中，行记录的第一部分倒序存放了一行数据中列的长度（Length），而 Redundant 中存的是每一列的偏移量（Offset），从总体上上看，Compact 行记录格式相比 Redundant 格式能够减少 20% 的存储空间。

MySQL要求一个行定义长度不能超过65535个字节（64KB），也就是说，**所有字段的长度加起来不能超过65535个字节**，text、blob等大字段类型除外。但是有一个问题，InnoDB一个页的默认大小为16KB，即16384字节，怎么能存放65535字节的数据呢？这是因为InnoDB可以将一条记录中的某些数据存储在真正的数据页之外，被称为行**行溢出数据**。

当 InnoDB 使用 Compact 或者 Redundant 格式存储极长的 VARCHAR 或者 BLOB 这类大对象时，并不会直接将所有的内容都存放在数据页节点中，而是将行数据中的前 768 个字节存储在数据页中，后面会通过偏移量指向溢出页。

![6](6.png)

还有一种更新的文件格式称为Barracuda，拥有两种新的行记录格式：Compressd和Dynamic。

新的两种行记录格式对于BLOB类型的数据采用了完全的行溢出存储方式，在数据页中只存放20个字节的指针，实际的数据都存放在Off Page中，而之前的Compact和Redundant会存放768个前缀字节。Compressd行记录会以zlib算法进行压缩，对于BLOB、TEXT、VARCHAR大长度类型的数据能够进行非常有效的压缩。

![7](7.png)