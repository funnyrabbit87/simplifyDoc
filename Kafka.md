# 文件物理存储结构
文件夹topic+partitionN，包含segment(段)数据文件中。但每个段segment file消息数量不一定相等，这种特性方便old segment file快速被删除。
- segment file组成：由2大部分组成，分别为index file和data file，此2个文件一一对应，成对出现，后缀”.index”和“.log”分别表示为segment索引文件、数据文件.
- segment文件命名规则：partion全局的第一个segment从0开始，后续每个segment文件名为上一个segment文件最后一条消息的offset值。数值最大为64位long大小，19位数字字符长度，没有数字用0填充。
![](/kafka_img/index-log-mapping.png)
以索引文件中元数据3,497为例，依次在数据文件中表示第3个message(在全局partiton表示第368772个message)、以及该消息的物理偏移地址为497。
查找过程，通过offset找到index和log文件，然后通过index找到最接近offset的记录，包含<offset,messageLength>, 然后从log文件中读取一批消息过滤到符合条件的消息。就是稀疏索引。