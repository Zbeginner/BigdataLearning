# CentOS6.8 安装Hue 4.2.0

`我也不知道为什么我这么坎坷。不知道还有谁会遇到我一样的问题。`

## 前提环境准备

环境：

```
python2.6.6  如果是python2.7.5+你会发现你的问题会少很多。但是奈何我是2.6.6，还不会改。
jdk1.8
maven3.6
mysql5.7
```

[Hue下载地址，里面找你想下载的版本就行了](https://github.com/cloudera/hue/releases)

```
1. 将jdk与maven配置到环境变量
2. yum -y install python-devel.x86_64 libffi libffi-devel gcc gcc-c++ kernel-devel openssl-devel libxslt-devel gmp-devel sqlite-devel openldap-devel 
```

### 安装

解压hue

cd hue

make apps

`以上的官网的介绍的安装教程就这么简单。但是问题还是很多`

报错：

PEP440Warning: 'logilab (astng-0.20.1)' is being parsed as a legacy, non PEP 440, version. You may find odd behavior and sort order. In particular it will be sorted as less than 0.0. It is recommend to migrate to PEP 440 compatible versions

make apps error : No local packages or download links found for logilab-astng>=0.24.3   

往上看会发现还有一堆问题，但是就是这个python库的问题。这个库在Python2.7.5以上的版本中pip一下就好(亲测成功)。但是2.6.6pip一下我这并没成功(<font color="red">自己升级一个python2.7.5或以上版本,只是做个软连接将/usr/bin/python指向你新装的python是不行的。因为hue解压的时候会去找原装的那个。我也不知道怎么解决</font>)

解决方式一(自动)：

```
pip install logilab-astng==0.24.3
```

能直接成功问题应该就能解决(我没成功，报了一堆错)。成功了就跳过方式二。

解决方式二(手动)：

[需要的依赖包](https://pan.baidu.com/s/1eeosDS-ZELORcKS1RAhr9Q)

密码:nywn

然后按照顺序解压安装

```
cd 解压文件
python setup.py install
```

顺序

```
linecache2-1.0.0.tar
six-1.12.0.tar
pbr-5.4.3.tar
traceback2-1.4.0.tar
unittest2-1.1.0.tar
logilab-common-0.53.0.tar
logilab-astng-0.24.3.tar
```

再到hue目录下make apps 就没问题了

```
1429 static files copied to '/opt/software/hue-release-4.2.0/build/static', 1429 post-processed.
make[1]: Leaving directory `/opt/software/hue-release-4.2.0/apps'
```

出现类似这样的结果就是成功了。

## 其它问题

如果中途因为网络问题，有什么包下不下来。直接复制网址自己下。然后手动装(解压、python setup.py intstall)

[如果装的途中有什么模块缺少去这个网站下载安装就好了](https://pypi.org/)

