# MR题目：如何一次Map读n行数据

## 输入数据

```json
{
"name":"ta",
"age":12,
"sex":1
}
{
"name":"la",
"age":13,
"sex":2
}
{
"name":"la",
"age":13,
"sex":2
}
{
"name":"la",
"age":13,
"sex":2
}
{
"name":"la",
"age":13,
"sex":2
}
{
"name":"la",
"age":13,
"sex":2
}
```

## 输出数据

```txt
{"name":"la","age":13,"sex":2}
{"name":"la","age":13,"sex":2}
{"name":"la","age":13,"sex":2}
{"name":"la","age":13,"sex":2}
{"name":"la","age":13,"sex":2}
{"name":"ta","age":12,"sex":1}
```

## 运行记录

```
Map-Reduce Framework
		Map input records=6   //从这里可以看出确实是一次读的五行
		Map output records=6
```

## 代码

### job

```java
public class JSONJob {
    static class JSONMapper extends Mapper<IntWritable, Text, Text, NullWritable> {
        @Override
        protected void map(IntWritable key, Text value, Context context) throws IOException, InterruptedException {
            context.write(value,NullWritable.get());
        }
    }

    public static void main(String[] args) throws IOException, ClassNotFoundException, InterruptedException {
        //1. 配置jbo
        Configuration con = new Configuration();

        Job job = Job.getInstance(con);

        //2. 指定程序jar包所在位置
        job.setJarByClass(JSONJob.class);
		// 重点是自定义读数据时的逻辑		
        job.setInputFormatClass(JsonInputFormat.class);


        //3. 配置Mapper和Reduce
        job.setMapperClass(JSONMapper.class);
        
        //4. 配置Mapper输出kv
        job.setMapOutputKeyClass(Text.class);
        job.setMapOutputValueClass(NullWritable.class);

        //5. 配置最终kv
        job.setOutputKeyClass(Text.class);
        job.setOutputValueClass(NullWritable.class);


        FileInputFormat.setInputPaths(job, new Path("d:/work/in/2"));
        FileOutputFormat.setOutputPath(job, new Path("d:/work/out/2"));

        //7. 提交代码
        boolean result = job.waitForCompletion(true);
        System.exit(result ? 0 : 1);
    }
}
```

### InputFormat

```java
//因为是文本，所以直接用了FileInputFormat来处理
public class JsonInputFormat extends FileInputFormat<IntWritable, Text> {
    //如果要切片，我没想到办法做到从JSON最后一个}切，所以直接不切片了，多大都进一个MapTask
    @Override
    protected boolean isSplitable(JobContext context, Path filename) {
        return false;
    }

    @Override
    public RecordReader<IntWritable, Text> createRecordReader(InputSplit split, TaskAttemptContext context) throws IOException, InterruptedException {
        return new FiveLineRecordReader(split, context);
    }
	//自定义RecordReader，读数据的逻辑
    class FiveLineRecordReader extends RecordReader<IntWritable, Text> {
        FileSplit fileSplit;  //切片信息
        Configuration conf;  //job配置信息
        Boolean progress;   //一个标志位，用来记录是否读到文件末尾的。false代表到了文件末尾
        IntWritable lineNum;  //按行读，记录的行号。类似默认的那个偏移量
        Text value; 		//存储读到的Json

        public FiveLineRecordReader(InputSplit fileSplit, TaskAttemptContext context) throws IOException, InterruptedException {
            initialize(fileSplit, context);
        }
		//初始化数据。无关紧要
        @Override
        public void initialize(InputSplit split, TaskAttemptContext context) throws IOException, InterruptedException {
            this.fileSplit = (FileSplit) split;
            conf = context.getConfiguration();
            lineNum = new IntWritable(0);
            progress = true;
            value = new Text();
        }
		
        //核心方法
        @Override
        public boolean nextKeyValue() throws IOException, InterruptedException {
            if (progress) {
                String path = fileSplit.getPath().toUri().getPath();
                path = path.substring(1);//我这里是本地跑的所以处理了以下。
                //1. 获取文件字符流
                BufferedReader bufferedReader = new BufferedReader(new InputStreamReader(new FileInputStream(path)));
                
                //2. 获取上一次读到的位置
                int lineNumNow = lineNum.get();
                
                //3. 跳过之前读过的行
                for (int i = 0; i < lineNumNow; i++) {
                    bufferedReader.readLine();
                }
                //3. 新读5行
                StringBuffer json = new StringBuffer();
                for (int i = 0; i < 5; i++) {
                    String tem = bufferedReader.readLine();
                    if (tem != null) {
                        json.append(tem);
                    } else break;
                }
                //4. 记录结果
                value.set(json.toString());
                lineNum.set(lineNum.get()+5);
                //5. 判断是否到文件末尾。
                if (bufferedReader.readLine() == null) {
                    progress = false;
                }
                return true;//nextKeyValue 返回true代表后面还有map数据
            }
            return false;
        }

        @Override
        public IntWritable getCurrentKey() throws IOException, InterruptedException {
            return lineNum;
        }

        @Override
        public Text getCurrentValue() throws IOException, InterruptedException {
            return value;
        }

        @Override
        public float getProgress() throws IOException, InterruptedException {
            return progress == true ? 0 : 1;
        }

        @Override
        public void close() throws IOException {

        }
    }
}
```