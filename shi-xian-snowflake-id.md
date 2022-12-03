# 实现 snowflake id



### 原理

```
  41bits  |   10bits   |     12bits
----------|------------|----------------
timestamp | machine id | sequence number
```

最早是twitter采用的，需要每秒生成数万不重复的id号。

特点 多结点部署 不重复 有序的 ID

> We currently use MySQL to store most of our online data. In the beginning, the data was in one small database instance which in turn became one large database instance and eventually many large database clusters. For various reasons, the details of which merit a whole blog post, we’re working to replace many of these systems with the Cassandra distributed database or horizontally sharded MySQL (using gizzard).

> Unlike MySQL, Cassandra has no built-in way of generating unique ids – nor should it, since at the scale where Cassandra becomes interesting, it would be difficult to provide a one-size-fits-all solution for ids. Same goes for sharded MySQL.

> Our requirements for this system were pretty simple, yet demanding:

> We needed something that could generate tens of thousands of ids per second in a highly available manner. This naturally led us to choose an uncoordinated approach.

> These ids need to be roughly sortable, meaning that if tweets A and B are posted around the same time, they should have ids in close proximity to one another since this is how we and most Twitter clients sort tweets.\[1]

> Additionally, these numbers have to fit into 64 bits. We’ve been through the painful process of growing the number of bits used to store tweet ids before. It’s unsurprisingly hard to do when you have over 100,000 different codebases involved.

翻译过来大概是说我们需要每秒产生数万个int64类型的id号，而且尽量的相近，没有更好的方案，于是我们自己解决了。

> To generate the roughly-sorted 64 bit ids in an uncoordinated manner, we settled on a composition of: timestamp, worker number and sequence number.

我们采用 时间戳 机器编号 顺序号 组成了一个 64 bit 的id号

> Snowflakes are 64 bits in binary. (Only 63 are used to fit in a signed integer.) The first 41 bits are a timestamp, representing milliseconds since the chosen epoch. The next 10 bits represent a machine ID, preventing clashes. Twelve more bits represent a per-machine sequence number, to allow creation of multiple snowflakes in the same millisecond. The final number is generally serialized in decimal.

### 二进制模拟结果

```
 111111111111111111111111111111111111111111111111111111111111111 int64 max
                             10111111111010011101001000000111100 timestamp
                                                               1 machineID
                                                               1 sequenceNumber

                                                               1 sequenceNumber
       101111111110100111010010000001111000000000000000000000000 timestamp shift
                                                   1000000000000 machineID shift
                                                               1 sequenceNumber
       101111111110100111010010000001111000000000001000000000001 id
                                              108037617659940865 id int64

```

之前我使用的是 [https://github.com/bwmarrin/snowflake](https://github.com/bwmarrin/snowflake) 的go实现版本，现在我实现了自己的版本，并更新到了langgo框架当中。

### 代码和视频

[https://github.com/langwan/chihuo/tree/main/go%E8%AF%AD%E8%A8%80/snowflake](https://github.com/langwan/chihuo/tree/main/go%E8%AF%AD%E8%A8%80/snowflake)

[https://www.bilibili.com/video/BV18P411A7pa/](https://www.bilibili.com/video/BV18P411A7pa/)

1. [https://en.wikipedia.org/wiki/Snowflake\_ID](https://en.wikipedia.org/wiki/Snowflake\_ID)
