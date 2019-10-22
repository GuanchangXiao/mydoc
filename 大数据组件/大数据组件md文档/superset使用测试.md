## superset使用测试

#### 1.安装及配置

##### 安装superset环境依赖

```
sudo yum upgrade python-setuptools
sudo yum install gcc gcc-c++ libffi-devel python-devel python-pip python-wheel openssl-devel libsasl2-devel openldap-devel
```

##### 安装python虚拟环境

```
pip install virtualenv
```

##### 安装python工具和pip工具

```
pip install --upgrade setuptools pip
```

##### 安装superset并启动

```
# Install superset
pip install superset

# Initialize the database
superset db upgrade

# Create an admin user (you will be prompted to set a username, first and last name before setting a password)
$ export FLASK_APP=superset
flask fab create-admin

# Load some data to play with
superset load_examples

# Create default roles and permissions
superset init

# To start a development web server on port 8088, use -p to bind to another port
superset run -p 8080 --with-threads --reload --debugger
```

#### 2.导入数据源与表

##### 点击sources，选择数据源如mysql，kylin以及CSV等。

![1571620832029](C:\Users\Ryan\AppData\Roaming\Typora\typora-user-images\1571620832029.png)

##### 填写数据源的配置信息，并测试连接

![1571621532832](C:\Users\Ryan\AppData\Roaming\Typora\typora-user-images\1571621532832.png)

##### 选择导入表，并填写对应信息，即可完成表的添加。

![1571621642735](C:\Users\Ryan\AppData\Roaming\Typora\typora-user-images\1571621642735.png)

##### **！！！Schema与Table_Name严格区分大小，否则会导入报错。**

##### 单击表名可自由配置查询表信息并生成图表

![1571622430992](C:\Users\Ryan\AppData\Roaming\Typora\typora-user-images\1571622430992.png)

#### 3.SQL查询

##### 打开SQL lab可以对数据源进行查询。

![1571622697899](C:\Users\Ryan\AppData\Roaming\Typora\typora-user-images\1571622697899.png)

#### 4.图表生成及配置

##### 选择charts并点击+号，创建一个新的图表。

![1571623118721](C:\Users\Ryan\AppData\Roaming\Typora\typora-user-images\1571623118721.png)

##### 配置图表查询条件及样式调整

![1571623380554](C:\Users\Ryan\AppData\Roaming\Typora\typora-user-images\1571623380554.png)

#### 5.dashboard生成及配置

##### 新建一个dashboard，并填写名称及所有者并导入图表

![1571624329399](C:\Users\Ryan\AppData\Roaming\Typora\typora-user-images\1571624329399.png)

**在构建dashboard时，存在bug。**

**新建的dashboard在导入图表时，导入失败，需要通过以下 方式导入。**

![1571624984022](C:\Users\Ryan\AppData\Roaming\Typora\typora-user-images\1571624984022.png)

**当dashboard中存在一个图表时，即可导入其他的图表**

