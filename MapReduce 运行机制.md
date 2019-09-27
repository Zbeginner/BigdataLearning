# MapReduce 运行机制

## 作业提交

- 作业切片

  客户端会先去拉取需要处理的数据文件，并对其切片(规划)，并生成一个**job.split**，文件里面记录着所有文件的切片信息。

  - 切片原理

    - 普通文件

      - 文件/切片大小<1.1直接一个切片
      - 文件大于/切片大小>1.1
        - 先一个个切片(就按照切片大小)
        - 最少一个切片按照切片1.1倍的大小切。

      ```java
      //如果可切割
      if (this.isSplitable(job, path)) {
          long blockSize = file.getBlockSize();
          //判断切片大小
          long splitSize = this.computeSplitSize(blockSize, minSize, maxSize);
      
          long bytesRemaining;
        int blkIndex;
          //先以一整个切片的大小切
        for(bytesRemaining = length; (double)bytesRemaining / (double)splitSize > 1.1D; bytesRemaining -= splitSize) {
              //如果块剩余大小大于切片的1.1倍，就以切片大小切一片。
              //这里也能看到切片其实是个逻辑，实际记录的也至少在实际数据块中的起始和结束位置。
            blkIndex = this.getBlockIndex(blkLocations, length - bytesRemaining);
              splits.add(this.makeSplit(path, length - bytesRemaining, splitSize, blkLocations[blkIndex].getHosts(), blkLocations[blkIndex].getCachedHosts()));
        }
      	//最后剩下的的一部份也开一个分片
        if (bytesRemaining != 0L) {
              blkIndex = this.getBlockIndex(blkLocations, length - bytesRemaining);
              splits.add(this.makeSplit(path, length - bytesRemaining, bytesRemaining, blkLocations[blkIndex].getHosts(), blkLocations[blkIndex].getCachedHosts()));
          }
      }
      ```
    
    - 数据库数据
    
      - 按行切
    
    - 不可切割的压缩文件(.zip)
    
      - 直接就算一个切片

- 规划完所有的切片后，收集jar包和运行的配置信息打包发给Yarn，提交作业请求。![client](img\client.png)

## Yarn(简略逻辑)

1. Yarn的ResourceManager收到作业信息后为它开一个**AppMaster**进程来管理资源的申请，任务的追踪，进度的监控。

 	2. 分配到资源后开启一个**Container**，拿着分配来的资源此时才开始一个运行一个切片任务

## Map

### 数据读入

​	之前提到过，切片其实是个逻辑过程，只是数据在某个文件块上的起始结束地址**SplitLocationInfo**。

​	而要拿到真的数据来给我们写的Mapper处理，就需要这样一个数据读入的过程。而这里用到的就是我们在**Driver中开启的FileInputFormat**。拿到的数据发给Mapper做下一步处理。

### Mapper

​	这阶段就是拿到数据后经过我们的Mapper处理数据。生成一个个K-V键值对发给下一阶段处理。如果我们没有自己写Mapper,自定义的Mapper就是文件原样输出。

### 数据写出

### OutputCollector

#### 环形缓冲区

 环形缓冲区(内存中)的概念就是能不断写的数据结构，数据不停的在写，然后前面写的数据会被写出到别的地方(如磁盘文件)，这样后面快写满了又可以回到前面来写。循环这样的过程就能不停的复用。

```java
//环形缓冲区默认100M(达到80%就会溢写)
int sortmb = this.job.getInt("mapreduce.task.io.sort.mb", 100);
int maxMemUsage = sortmb << 20;
this.kvbuffer = new byte[maxMemUsage]; //100M
```

#### 排序，合并(Map的shuffle)

 Map切出来的K-V对，会在一个环形缓冲区处理，然后如果整个环形缓冲区的使用大小到80%的时候，会发生一次溢写(写到磁盘)在时候会对这些一些出去的K-V进行排序、分区。读到数据最后再将环形缓冲区里的数据直接交给下一个阶段。

 如果切片很大会被溢出写到很多份文件内。

 一个切片任务最后生成的文件，如果是多份还会经历新一次合并、分区、排序的过程直到最后生产一个大文件交给Reduce处理。

## Reduce

 Reduce阶段的工作相对简单，就是对Mapper阶段生成的大文件进行分区(默认一个分区)然后排序，最后通过我们定义的OutputFormat写出。

## 思维导图

## ![MapReduce流程](img\MapReduce流程.png)

