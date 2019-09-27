# MapReduce自定义分区Partitioner

1. 需求：

   统计电话号码，要求按归属地放入不同的文件中。这样的需求就会用到分区。

   ```text
   //我们以138、137、136开头的分一个区，以其它数字开头的分另外一个区
   13899999
   13888888
   13877777
   13699999
   13688888
   13677777
   13799999
   13788888
   13777777
   14899999
   14888888
   14877777
   15299999
   15288888
   15277777
   ```

2. 如何分区

   1. 自定义Partitioner

      ```java
      public class MyPartitioner extends Partitioner {
          //参数o为key
          //o2为value
          //i为numReduceTasks
          public int getPartition(Object o, Object o2, int i) {
              String key=o.toString();
              if(key.startsWith("138")){
                  return 0;
              }else if(key.startsWith("137")){
                  return 1;
              }else if(key.startsWith("136")){
                  return 2;
              }
              return 3;
          }
      }
      ```

   2. 绑定job信息

   ```java
   //绑定自定义Partitioner
   job.setPartitionerClass(MyPartitioner.class);
   //这相当于设置会输出几个文件
   job.setNumReduceTasks(4);
   ```

3. 完整代码

   ```java
   public class MyDriver {
       public static class MyMapper extends Mapper<LongWritable, Text, Text, NullWritable> {
           @Override
           protected void map(LongWritable key, Text value, Context context) throws IOException, InterruptedException {
               context.write(value,NullWritable.get());
           }
       }
   
       public static class MyReducer extends Reducer<Text, NullWritable, Text, NullWritable> {
           @Override
           protected void reduce(Text key, Iterable<NullWritable> values, Context context) throws IOException, InterruptedException {
               context.write(key, NullWritable.get());
           }
       }
   
       public static void main(String[] args) throws IOException, ClassNotFoundException, InterruptedException {
           //1. 配置jbo
           Configuration con = new Configuration();
   
           Job job = Job.getInstance(con);
   
           job.setPartitionerClass(MyPartitioner.class);
           job.setNumReduceTasks(4);
           //2. 指定程序jar包所在位置
           job.setJarByClass(MyDriver.class);
   
           //3. 配置Mapper和Reduce
           job.setMapperClass(MyMapper.class);
           job.setReducerClass(MyReducer.class);
   
           //4. 配置Mapper输出kv
           job.setMapOutputKeyClass(Text.class);
           job.setMapOutputValueClass(NullWritable.class);
   
           //5. 配置最终kv
           job.setOutputKeyClass(Text.class);
           job.setOutputValueClass(NullWritable.class);
   
           //6. 配置输入输出文件位置（本地模式跑的）
           FileInputFormat.setInputPaths(job, new Path("d:/work/in/partitioner"));
           FileOutputFormat.setOutputPath(job, new Path("d:/work/out7"));
           
           //7. 提交代码
           boolean result = job.waitForCompletion(true);
           System.exit(result ? 0 : 1);
       }
   }
   ```

   ```java
   public class MyPartitioner extends Partitioner {
       //参数o为key
       //o2为value
       //i为numReduceTasks
       public int getPartition(Object o, Object o2, int i) {
           String key=o.toString();
           if(key.startsWith("138")){
               return 0;
           }else if(key.startsWith("137")){
               return 1;
           }else if(key.startsWith("136")){
               return 2;
           }
           return 3;
       }
   }
   ```

4. 注意到

   1. setNumReduceTasks的数量要么等于或大于你的分区数，要么为1，否则无法正常分区