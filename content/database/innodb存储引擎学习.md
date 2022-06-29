---
title: "innodb存储引擎学习"
date: 2021-03-25T12:27:39+08:00
draft: false
toc: true
---

数据库软件的作用就是保存数据，并提供接口查询数据（sql select）。数据库保管的数据肯定是要以**文件**的形式持久化到磁盘上，查询数据的时候肯定要从磁盘读取文件取出数据。要理解数据库的原理就是要搞清楚数据库保存数据的文件的格式，以及取数据时又是如何快速的取出数据。

# 准备工作

- 建立一个数据库：

```sql
creare database test;
```

创建一个数据库后，mysql会在其数据目录（datadir）建立一个和数据库名同名的目录。数据目录是那个目录呢？可以用如下命令查看：

```sql
MariaDB [test]> show variables like 'datadir';
+---------------+-----------------+
| Variable_name | Value           |
+---------------+-----------------+
| datadir       | /var/lib/mysql/ |
+---------------+-----------------+
1 row in set (0.001 sec)
```

在执行`creare database test;`后，`/var/lib/mysql/ `会新建目录`test`:

```txt
vagrant@ubuntu:/var/lib/mysql$ tree -F -L 1 /var/lib/mysql/
/var/lib/mysql/
├── aria_log.00000001
├── aria_log_control
├── debian-10.3.flag
├── ib_buffer_pool
├── ib_logfile0
├── ib_logfile1
├── ibdata1
├── ibtmp1
├── multi-master.info
├── mysql/
├── mysql_upgrade_info
├── performance_schema/
├── tc.log
└── test/
```

test目录里只有一个名为`db.opt`的文件。

- 建立一个表：

```sql
use test;
CREATE TABLE `t_demo` (
  `id` int(11) NOT NULL AUTO_INCREMENT,
  `c1` int(11) DEFAULT NULL,
  `c2` int(11) NOT NULL,
  `c3` varchar(16) DEFAULT NULL,
  PRIMARY KEY (`id`),
  UNIQUE KEY `c1` (`c1`),
  KEY `c2` (`c2`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
```

执行建表语句后，test目录会多一些文件：

```txt
root@ubuntu:/var/lib/mysql/test# ls -al
total 148
drwx------ 2 mysql mysql   4096 Mar 31 14:33 .
drwxr-xr-x 5 mysql mysql   4096 Mar 26 12:58 ..
-rw-rw---- 1 mysql mysql     67 Mar  5 11:58 db.opt
-rw-rw---- 1 mysql mysql   2033 Mar 31 14:33 t_demo.frm
-rw-rw---- 1 mysql mysql 131072 Mar 31 14:33 t_demo.ibd
```

多了2个文件：`t_demo.frm`和`t_demo.ibd`。`t_demo.frm`是表结构的定义文件，而`t_demo.ibd`就是存放数据的文件，（索引也是存在这个文件中）。

# 什么是页

刚刚新建的表，t_demo.ibd文件的大小是131072字节，也就是128KB。其实innodb存储引擎是以**页**（page）为单位来组织数据的。一个page的大小是16KB，因此新建的表的数据文件t_demo.ibd含有8个页。（没有索引时，新建的表数据文件只有96KB，因为我们的表除了主键还有2个辅助索引，因此初始的时候就多了2个页面，对应2个索引页）。

页有很多种不同的类型，不同的页的16KB的结构是有区别的，但是前38个字节的结构是一样的，叫做“File Header”。结构如下：

| 偏移 | 占用字节数 | 描述                                                          |
| ---- | ---------- | ------------------------------------------------------------- |
| 0    | 4          | 校验和                                                        |
| 4    | 4          | 页号                                                          |
| 8    | 4          | 上一个页号                                                    |
| 12   | 4          | 下一个页号                                                    |
| 16   | 8          | 页面被最后修改的LSN值                                         |
| 24   | 2          | 页面的类型                                                    |
| 26   | 8          | 仅在系统表空间的第一页定义，表示文件至少被刷新到了对应的LSN值 |
| 34   | 4          | 页面属于那个表空间                                            |

很明显，`页号`,`上一个页号`,`下一个页号`这3个字段可以使得若干页面在逻辑上形成一个**双向链表**,事实上innodb以B+树的形式组织数据和索引，B+树的每一个节点就是一个页，B+树的同一层的节点构成双向链表。

我们可以写一个简单的python3程序，将初创建的表空间文件t_demo.ibd中的8个页的字段关注字段打印出来。

python3程序：

innodb_page_info.py
```python
#!/usr/bin/env python3
# coding=utf-8

import os
import sys
from struct import unpack

PAGE_SIZE = 16 * 1024


def parse_file(file_name):
    file_size = os.path.getsize(file_name)
    assert file_size % PAGE_SIZE == 0
    page_count = file_size / PAGE_SIZE
    print("There is %d pages in innodb table space file %s" % (page_count, file_name))
    count = 0
    with open(file_name, 'rb') as f:
        while count < page_count:
            page = f.read(PAGE_SIZE)
            parse_page(page)
            count += 1


def parse_page(page):
    page_type = unpack('>H', page[24:26])[0]
    page_num = unpack('>i', page[4:8])[0]
    page_prev = unpack('>i', page[8:12])[0]
    page_next = unpack('>i', page[12:16])[0]
    print("page_type = %-8s page_num = %-8s page_prev = %-8s page_next = %-8s" %
          (hex(page_type), page_num, page_prev, page_next))


if __name__ == '__main__':
    fn = sys.argv[1]
    parse_file(fn)
```

以文件`t_demo.ibd`的绝对路径作为参数运行程序，程序的输出为：

```txt
zk@zk-mkp:~/code/innodb_page_info » python3 innodb_page_info.py /Users/zk/tmp/t_demo.ibd
There is 8 pages in innodb table space file /Users/zk/tmp/t_demo.ibd
page_type = 0x8      page_num = 0        page_prev = 0        page_next = 0
page_type = 0x5      page_num = 1        page_prev = 0        page_next = 0
page_type = 0x3      page_num = 2        page_prev = 0        page_next = 0
page_type = 0x45bf   page_num = 3        page_prev = -1       page_next = -1
page_type = 0x45bf   page_num = 4        page_prev = -1       page_next = -1
page_type = 0x45bf   page_num = 5        page_prev = -1       page_next = -1
page_type = 0x0      page_num = 0        page_prev = 0        page_next = 0
page_type = 0x0      page_num = 0        page_prev = 0        page_next = 0
```

page_type可以有如下几种值：

| 十六进制值 | 描述                    |
| ---------- | ----------------------- |
| 0X0000     | 未使用                  |
| 0X0002     | undo日志页              |
| 0X0003     | 存储段信息的页          |
| 0X0004     | Change Buffer空闲列表   |
| 0X0005     | Change Buffer的一些属性 |
| 0X0006     | 存储一些系统属性的页    |
| 0X0007     | 存储事务系统数据的页    |
| 0X0008     | 表空间头部信息          |
| 0X0009     | 存储区信息的页          |
| 0X000A     | 溢出页                  |
| 0X45BF     | 索引页，也就是数据页    |

可以看到，目前前3个页用到了8，5，3这3种页，接着是3个数据页（类型是0X45BF）。这3个数据页分别是：

- 主键B+树的根节点
- c1列索引的B+树根节点
- c2列索引的B+树的根节点

因为表才刚刚建好，还一条数据都没有，所以，3棵B+树都只有1层，这个层只有一个根节点。没有前一页，也没有后一页，所以page_prev和page_next都是-1

# 准备测试数据

为了简单演示innodb如何从表空间文件`t_demo.idb`中查询到指定的数据，我们需要往t_demo表中插入一些数据。

写一个脚本来完成这这件事，先写个用来生成csv文件的python脚本：

create_demo_data.py
```python
#!/usr/bin/env python3
# coding=utf-8
import sys

def create_csv_file(file_name):
    with open(file_name, 'w') as f:
        for count in range(1, 2001):
            _id = count
            c1 = count
            c2 = 2001 - count
            c3 = hex(count)
            f.write('%d, %d, %d, "%s"\n' % (_id, c1, c2, c3))


if __name__ == '__main__':
    fn = sys.argv[1]
    create_csv_file(fn)
```

执行`python3 create_demo_data.py a.txt`生成a.txt文件，然后使用如下的语句导入数据：

```sql
load data local infile '/tmp/a.txt' into table t_demo fields terminated by ',' optionally enclosed by '"' escaped by '"' lines terminated by '\n';
```

导入数据后，表空间文件肯定会变大，在这个例子里有304KB，一共19个页：

```txt
root@ubuntu:/var/lib/mysql/test# clear
root@ubuntu:/var/lib/mysql/test# ls -lh
total 312K
-rw-rw---- 1 mysql mysql   67 Mar  5 11:58 db.opt
-rw-rw---- 1 mysql mysql 2.0K Mar 31 15:39 t_demo.frm
-rw-rw---- 1 mysql mysql 304K Mar 31 15:39 t_demo.ibd
```

用前面的`innodb_page_info.py`解析这个304KB的文件，得到输出：

```txt
There is 19 pages in innodb table space file /Users/zk/tmp/t_demo.ibd
page_type = 0x8      page_num = 0        page_prev = 0        page_next = 0
page_type = 0x5      page_num = 1        page_prev = 0        page_next = 0
page_type = 0x3      page_num = 2        page_prev = 0        page_next = 0
page_type = 0x45bf   page_num = 3        page_prev = -1       page_next = -1
page_type = 0x45bf   page_num = 4        page_prev = -1       page_next = -1
page_type = 0x45bf   page_num = 5        page_prev = -1       page_next = -1
page_type = 0x45bf   page_num = 6        page_prev = -1       page_next = 7
page_type = 0x45bf   page_num = 7        page_prev = 6        page_next = 8
page_type = 0x45bf   page_num = 8        page_prev = 7        page_next = 9
page_type = 0x45bf   page_num = 9        page_prev = 8        page_next = 14
page_type = 0x45bf   page_num = 10       page_prev = -1       page_next = 11
page_type = 0x45bf   page_num = 11       page_prev = 10       page_next = 15
page_type = 0x45bf   page_num = 12       page_prev = 17       page_next = 13
page_type = 0x45bf   page_num = 13       page_prev = 12       page_next = -1
page_type = 0x45bf   page_num = 14       page_prev = 9        page_next = 16
page_type = 0x45bf   page_num = 15       page_prev = 11       page_next = -1
page_type = 0x45bf   page_num = 16       page_prev = 14       page_next = -1
page_type = 0x45bf   page_num = 17       page_prev = -1       page_next = 12
page_type = 0x0      page_num = 0        page_prev = 0        page_next = 0
```

可以看出3个双向链表：

`6 <-> 7 <-> 8 <-> 9 <-> 14 <-> 16`

`10 <-> 11 <-> 15`

`17 <-> 12 <-> 13`

这个3个双向链表应该是3棵B+树的第2层，前面说过了3棵B+树的根节点分别是页3，页4，页5，那怎么知道3个根和3个链表如何对应的呢？这个就需要用到“Page Header”了。当页面类型是“0X45BF”时，从38字节开始后续的56个字节是“Page Header”，这56个字节的意义依次如下：

| 偏移 | 字节数 | 描述                                                              |
| ---- | ------ | ----------------------------------------------------------------- |
| 38   | 2      | 页目录槽数量                                                      |
| 40   | 2      | 还未使用的最小地址                                                |
| 42   | 2      | 第1个bit表示是否是紧凑类型的记录，甚于15个bit表示本页中的记录数量 |
| 44   | 2      | 已删除的记录构成的单链表的头节点偏移                              |
| 46   | 2      | 已删除的记录占用的字节数                                          |
| 48   | 2      | 最后插入记录的位置                                                |
| 50   | 2      | 记录插入的方向                                                    |
| 52   | 2      | 一个方向连续插入的记录数量                                        |
| 54   | 2      | 该页中的用户记录数                                                |
| 56   | 8      | 修改当前页的最大事务id，仅在二级索引页面中定义                    |
| 64   | 2      | 当前页在B+数中的层级                                              |
| 66   | 8      | 索引id，表示当前页属于那个索引                                    |
| 74   | 10     | B+树叶子段的头部信息，仅在B+树的根页面中定义                      |
| 84   | 10     | B+树非叶子段的头部信息，仅在B+树的根页面中定义                    |


修改下前面的innodb_page_info.py脚本为innodb_page_info2.py,将page header中的部分信息打印出来(只过滤出类型为0x45BF的页)

innodb_page_info2.py
```python
#!/usr/bin/env python3
# coding=utf-8

import os
import sys
from struct import unpack

PAGE_SIZE = 16 * 1024


def parse_file(file_name):
    file_size = os.path.getsize(file_name)
    assert file_size % PAGE_SIZE == 0
    page_count = file_size / PAGE_SIZE
    print("There is %d pages in innodb table space file %s" % (page_count, file_name))
    count = 0
    with open(file_name, 'rb') as f:
        while count < page_count:
            page = f.read(PAGE_SIZE)
            parse_page(page)
            count += 1


def parse_page(page):
    page_type = unpack('>H', page[24:26])[0]
    if page_type != 0x45bf:
        return
    page_num = unpack('>i', page[4:8])[0]
    page_prev = unpack('>i', page[8:12])[0]
    page_next = unpack('>i', page[12:16])[0]

    record_count = unpack('>H', page[54:56])[0]
    tree_level = unpack('>H', page[64:66])[0]
    index_id = unpack('>Q', page[66:74])[0]

    print("page_type = %-8s page_num = %-8s page_prev = %-8s page_next = %-8s record_count = %-5s tree_level = %-5s index-id = %-5s" %
          (hex(page_type), page_num, page_prev, page_next, record_count, tree_level, index_id))


if __name__ == '__main__':
    fn = sys.argv[1]
    parse_file(fn)
```

以相应的`t_demo.ibd`文件为参数运行此python3程序，输入大致如下：

```txt
There is 19 pages in innodb table space file /Users/zk/tmp/t_demo.ibd
page_type = 0x45bf   page_num = 3        page_prev = -1       page_next = -1       record_count = 6     tree_level = 1     index-id = 36
page_type = 0x45bf   page_num = 4        page_prev = -1       page_next = -1       record_count = 3     tree_level = 1     index-id = 37
page_type = 0x45bf   page_num = 5        page_prev = -1       page_next = -1       record_count = 3     tree_level = 1     index-id = 38
page_type = 0x45bf   page_num = 6        page_prev = -1       page_next = 7        record_count = 191   tree_level = 0     index-id = 36
page_type = 0x45bf   page_num = 7        page_prev = 6        page_next = 8        record_count = 377   tree_level = 0     index-id = 36
page_type = 0x45bf   page_num = 8        page_prev = 7        page_next = 9        record_count = 376   tree_level = 0     index-id = 36
page_type = 0x45bf   page_num = 9        page_prev = 8        page_next = 14       record_count = 376   tree_level = 0     index-id = 36
page_type = 0x45bf   page_num = 10       page_prev = -1       page_next = 11       record_count = 560   tree_level = 0     index-id = 37
page_type = 0x45bf   page_num = 11       page_prev = 10       page_next = 15       record_count = 1120  tree_level = 0     index-id = 37
page_type = 0x45bf   page_num = 12       page_prev = 17       page_next = 13       record_count = 1166  tree_level = 0     index-id = 38
page_type = 0x45bf   page_num = 13       page_prev = 12       page_next = -1       record_count = 602   tree_level = 0     index-id = 38
page_type = 0x45bf   page_num = 14       page_prev = 9        page_next = 16       record_count = 376   tree_level = 0     index-id = 36
page_type = 0x45bf   page_num = 15       page_prev = 11       page_next = -1       record_count = 320   tree_level = 0     index-id = 37
page_type = 0x45bf   page_num = 16       page_prev = 14       page_next = -1       record_count = 304   tree_level = 0     index-id = 36
page_type = 0x45bf   page_num = 17       page_prev = -1       page_next = 12       record_count = 232   tree_level = 0     index-id = 38
```

tree_level为1个3个页，说明是非叶子节点，记录数分别为6，3，3，刚好对应前面3个双向链表的长度。从这个输出页可以看出每个tree_level为0的叶子点页中的记录数。

更进一步，只要了解到全部页的二进制结构和逻辑关系，可以用python脚本分析页中的数据，模拟sql执行的过程，通过某个字段的索引定位到某个记录项，然后用主键id到聚簇索引执行回表操作。
