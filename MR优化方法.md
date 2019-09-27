# MapReduce 优化

## MR效率的瓶颈

1. 计算机硬件性能
2. I/O操作
   1. 数据倾斜
   2. map和reduce数设置不合理
   3. map运行时间太长，导致reduce等待过久
   4. 小文件过多
   5. 大量的不可分块的超大文件
   6. spill次数太多
   7. merge次数过多

## MR优化的方法

### 数据的输入

1. 合并小文件：在执行mr任务之前将小文件合并，大量的小文件会产生大量的map任务，增大map任务装载次数，而任务的装载比较耗时，从而导致mr运行较慢
2. 采用CombineTextFileInputFormat来作为输入，解决输入端大量小文件的场景

### Map阶段

1. 减少溢写的次数：通过调整io.sort.mb及sort.spill.percent参数值，增大触发spill的内存上限。减少磁盘IO。
2. 减少合并的次数：通过调整io.sort.factor参数，增大merge的文件数目，减少merge的次数，从而缩短mr处理的时间
3. 在map之后，不影响业务逻辑的前提下，先进行combine处理，减少IO。

### Reduce阶段

1. 合理设置map和reduce数：两个都不能设置太少，也不能设置太多。太少，会导致task等待，延迟处理时间；太多，会导致map、reduce任务竞争资源，造成处理超时等错误。
2. 设置map、reduce共存：调整slowstart.completedmaps参数，使map运行到一定程度后，reduce也开始运行，减少reduce的等待时间。
3. 规避使用reduce
4. 合理设置reduce端的buffer:默认情况下，数据达到一定阈值的时候，buffer中的数据就会写入磁盘，然后reduce会从磁盘中获得所有的数据。

### IO传输

1. 采用数据压缩的方式，减少网络IO的时间。
2. 使用SequenceFile二进制文件

### 数据倾斜问题

1. 数据倾斜现象
   1. 数据频率倾斜——某一个区域的数据量要远远大于其他区域
   2. 数据大小倾斜——部分记录的大小远远大于平均值
2. 减少数据倾斜的方法
   1. 抽样和范围分区
   2. 自定义分区
   3. combine
   4. 采用map join 尽量避免reduce join



