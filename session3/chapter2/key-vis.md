# 2.1 识别集群热点和业务访问模式

TiDB Dashboard 通过 Key Visualizer (KeyVis) 工具提供了一个简单，直观，符合直觉的方式来帮助用户观测 TiDB 的数据访问模式以及数据热点，
第一次让 DBA 或者开发者可以从数据库的角度去理解业务的负载，看到业务负载大概的长相，有点像医学上的 CT 成像，
从而能根据负载的模式给业务的开发人员提供很好的改进建议，改进业务系统的设计。

## 1. 概念

- Bucket

  一个 TiDB 集群中，Region（即一个数据块） 的数量可能多达数十万。一般难以将这么多的 Region 信息
  显示在一个屏幕上。因此，在一张热力图中，Region 会被压缩到约 1500 个连续范围，每个范围称为「Bucket」。
  Region 压缩总是倾向于将流量较小的大量 Region 压缩为一个 Bucket，而尽量让高流量的 Region 独立成 Bucket，
  以便突出热点情况。

- 热力图

  热力图是热点可视化的核心，它显示了一个指标随时间的变化。热力图的横轴 X 是时间，纵
  轴 Y 则表示集群里面的 Region，横跨 TiDB 集群上所有数据库和数据表。颜色越暗（cold）
  表示该区域的 Region 在这个时间段上读写流量较低，颜色越亮（hot）表示读写流量越高，即越热。

## 2. 界面图示

借用如下的热点示例图，可以观察到以下信息：

- 一个大型热力图，显示访问流量随时间的变化情况
- 下方和右侧为沿热力图的每个轴的平均值
- 左侧为表名、索引名等信息

![overview](res/session3/chapter2/keyvis/overview.png)

## 3. 观察某一段时间或者 Region 范围

热点可视化默认会显示最近 6 小时整个数据库内的热力图。其中，越靠近右侧（当前时间）
时，每个 Bucket 对应的时间间隔会越小。如果您想观察某个特定时间段或者特定的 Region
范围，则可以通过放大来获得更多细节。

- 在热图中向上或向下滚动
- 点击 Select & Zoom 按钮，然后点击并拖动以选择要放大的区域
- 点击 Reset 按钮，将 Region 范围重置为整个数据库
- 点击 时间选择框，重新选择观察时间段

注：使用后几种方法，将引起热图的重新绘制，您可能观察到热图与放大前有较大差异，这是
一个正常的现象。它可能是由于在进行局部观察时，Region 压缩的粒度发生了变化，或者是
局部范围内，「热」的基准发生了改变。

## 4. 调整亮度

热图使用颜色的明暗来表达一个 Bucket 的流量高低，颜色越暗（cold）表示该区域的 Region
在这个时间段上读写流量较低，颜色越亮（hot）表示读写流量越高，即越热。如果热图中的
颜色太亮或太暗，则可能很难观察到细节。此时，我们可以点击 Brightness 按钮，然后通过
滑块来调节页面的亮度。
注：显示一个区域内的热图时，会根据区域内的流量情况来界定冷热。当整个区域流量不够
均匀时，即使整体流量在数值上很低，您依然有可能观察到较大的亮色区域。请注意一定要结合
数值一起分析。

## 5. 自动刷新

当我们需要实时观察数据库的流量分布情况时，可以点击 Auto Refresh 按钮来让热图每分钟
自动刷新。请注意，如果您进行了时间范围或者 Region 范围的调整，自动刷新会被关闭。

## 6. 选择不同的指标

可以通过「指标选择框」来查看您关心的指标，目前支持如下指标：

- Read (bytes) 读流量
- Write (bytes) 写流量
- Read (keys) 读取行数
- Write (keys) 写入行数
- All 读写流量的总和

## 7. 关于热点引发的问题

热点的本质是大多数读写流量都只涉及个别 Region，进而导致集群中只有个别 TiKV 节点承载了大部分操作。KeyVis 将所有 Region 的读写流量按时间依次展示出来，使用颜色明暗表示读写流量的多少，以热力图的方式呈现。
通常业务在访问数据库的时候都有特定的模式，而数据库的监控系统往往难以识别这些模式，需要数据库专家或者资深的 DBA 靠自己多年丰富的经验仔细的推测。所有业务开发人员可能会经常被问到一些问题，比如：

- 读多写少还是写多读少？
- 读写比例是怎样的？
- 有没有热点？
- 追加写还是随机写？
- 一直均匀的读写还是有周期性流量（比如每小时一次促销）？
- 有没有突发性流量？
- 如果业务发展了，是否存在设计瓶颈，能不能 scale 10 倍？
- 有没有扫表(其实就是顺序读)？

业务开发人员面对这些问题有时候一脸问号，脑子里面快速过了一遍自己的业务，大概有一百多张表，每个都有增删改查（CRUD），还有各种统计汇总的业务，有很多请求还是用户触发的，这些问题真的是很难回答。但是现在通过 TiDB Dashboard 很多业务的访问模式都可以得到比较直观的观测， DBA 和开发者能够一眼发现系统的读写比例，哪里有热点，甚至能够精确到哪个数据块，哪些索引，也能预测如果流量持续增长，数据库系统是否能够轻松扩展。

## 8. 几种常见的访问模式

- 顺序读写

  这里是一个三张表的顺序写入的截图，用的 (sysbench prepare 命令)最左边我们可以看到表名为 sbtest1, sbtest2, sbtest3, 注意观察黄色（最亮的）的区块，大体上是呈现一个向上的斜线，这里的颜色亮度越高代表数据写得越多

  ![sequential-write](res/session3/chapter2/keyvis/sequential.png)

- 持续热点

  通常如果业务有持续的热点，在 KeyVis 上也是一眼就看出来了，如下图，可以看到连续的金黄色光条，每个光条代表一段持续的热点块

  ![hot](res/session3/chapter2/keyvis/hot.png)

如果我们把光标移动到金黄色的热点上还能看到更具体的提示，如下图所示，可以看到每分钟的流量为 165.25 兆字节, 访问的表名是 sbtest1， 访问的对象是表的一个索引，索引的名字是 k_1, 我们也能看到这个块在存储层对应的 key 范围，即图中展示的 start key 和 end key。

![tooltip](res/session3/chapter2/keyvis/tooltip.png)

- 均匀分布

  相信聪明的同学已经猜出来了，没有明显热点就是均匀负载，有兴趣的同学可以自己动手尝试构造一个类似的负载。这种负载下数据库有完美的扩展性，随着业务的流量上升做好扩容操作即可，流量下来了直接缩容。

  ![well-dist](res/session3/chapter2/keyvis/well-dist.png)

- 周期性负载

  系统的负载每隔一段时间会上来一次，随后又下降，如此反复，比如定时任务，整点促销，这些从图上看一目了然

  ![periodic](res/session3/chapter2/keyvis/periodic.png)

如果大家用的是 TiDB 4.0 或者以上的版本，系统会自动根据负载情况来弹性伸缩，不再需要人工干预。弹性伸缩相关的细节请参考弹性调度章节。
