#ElasticSearch（一）安装

**1、下载安装包:**

Linux: [elasticsearch-7.3.1-linux-x86_64.tar.gz](https://artifacts.elastic.co/downloads/elasticsearch/elasticsearch-7.3.1-linux-x86_64.tar.gz)

```shell
curl -L -O https://artifacts.elastic.co/downloads/elasticsearch/elasticsearch-7.3.1-linux-x86_64.tar.gz
```

macOS: [elasticsearch-7.3.1-darwin-x86_64.tar.gz](https://artifacts.elastic.co/downloads/elasticsearch/elasticsearch-7.3.1-darwin-x86_64.tar.gz)

```shel
curl -L -O https://artifacts.elastic.co/downloads/elasticsearch/elasticsearch-7.3.1-darwin-x86_64.tar.gz
```

Windows: [elasticsearch-7.3.1-windows-x86_64.zip](https://artifacts.elastic.co/downloads/elasticsearch/elasticsearch-7.3.1-windows-x86_64.zip)

**2、解压安装包:**

Linux:

```shell
tar -xvf elasticsearch-7.3.1-linux-x86_64.tar.gz
```

macOS:

```shell
tar -xvf elasticsearch-7.3.1-darwin-x86_64.tar.gz
```

Windows PowerShell:

```shell
Expand-Archive elasticsearch-7.3.1-windows-x86_64.zip
```

------

**3、启动ES**

Linux and macOS:

```shell
cd elasticsearch-7.3.1/bin
./elasticsearch
```

Windows:

```shell
cd elasticsearch-7.3.1\bin
.\elasticsearch.bat
```

**4、启动ES集群（可选）**

Linux and macOS:

```shell
./elasticsearch -Epath.data=data2 -Epath.logs=log2
./elasticsearch -Epath.data=data3 -Epath.logs=log3
......
```

Windows:

```shell
.\elasticsearch.bat -E path.data=data2 -E path.logs=log2
.\elasticsearch.bat -E path.data=data3 -E path.logs=log3
......
```