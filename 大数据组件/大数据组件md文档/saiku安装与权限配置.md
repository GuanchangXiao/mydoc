##### saiku安装及权限配置

### 1、安装配置

安装配置及运行不作过多讲解，详情参照kylin+saiku官网文档

[kylin官网推荐]: http://kylin.apache.org/blog/2019/05/23/saiku-kylin-molap/	"kylin官网推荐用法"

#### 1.1、安装运行中常见疑问

##### 注册证书

[注册地址]: https://licensing.meteorite.bi/login	"注册地址"

请务必认真填写邮箱地址，后续注册成功后登录需要从邮箱验证地址访问才能登录。

##### 上传证书

 访问 http://localhost:8080/upload.html 上传 License。上传所需用户名与密码均为 admin 

##### 登录访问

访问 http://localhost:8080 进行登录，用户名密码均为 admin 

##### 端口冲突解决办法

在saiku安装目录内的tomcat/conf中修改server.xml文件，一般冲突端口8080、8005

修改完成后如果依旧不能启动。

查看saiku目录启动的java进程是否在运行，如果有，杀掉

然后重启。

```
ps -ef|grep java  
kill -9  pid 
```

### 2、权限配置

##### 添加用户

打开控制台，选择Add  User

认真填写相关信息，***角色后续会用到***

![(C:\Users\Ryan\AppData\Roaming\Typora\typora-user-images\image-20191025133942374.png)

![image-20191025134347562](C:\Users\Ryan\AppData\Roaming\Typora\typora-user-images\image-20191025134347562.png)

##### 配置数据源

打开repository，找到datasources目录，发现该目录是空的的

1、创建数据源文件夹，文件夹名称与自己导入的数据源名称一致。

2、创建数据源下的 cube文件夹，文件夹名称与cube名称一致。

3、对文件夹进行权限配置 ，点击右边人头图标，配置角色及权限。

4、角色与创建用户时设置的角色名保持一致。

![image-20191025134849692](C:\Users\Ryan\AppData\Roaming\Typora\typora-user-images\image-20191025134849692.png)

##### 设置角色和权限

注意这里填写的角色名称与之前创建用户的角色名称要一一对应

![image-20191025135025064](C:\Users\Ryan\AppData\Roaming\Typora\typora-user-images\image-20191025135025064.png)

##### 修改数据源安全协议

打开数据源配置界面，security栏，选择one to one mapping，这步顺序不重要

![image-20191025135146502](C:\Users\Ryan\AppData\Roaming\Typora\typora-user-images\image-20191025135146502.png)

##### 登录不同角色账号访问不同cube

退出admin账号，登录配置账号验证权限。

异常查询可以查看日志，saiku安装目录/tomcat/logs/saiku.log

```
DEBUG [org.saiku.web.service.SessionService] Logging in with [org.springframework.security.core.userdetails.User@fd3765a: Username: xiaoshou; Password: [PROTECTED]; Enabled: true; AccountNonExpired: true; credentialsNonExpired: true; AccountNonLocked: true; Granted Authorities: ROLE_USER,ROLE_XS]
2019-10-25 13:18:27,847 INFO  [org.saiku.web.core.SecurityAwareConnectionManager] Setting role to datasource:kylin_learn role:ROLE_XS
2019-10-25 13:18:27,848 INFO  [org.saiku.web.core.SecurityAwareConnectionManager] Setting role to datasource:kylin_learn role:ROLE_XS
2019-10-25 13:18:28,016 INFO  [org.saiku.web.core.SecurityAwareConnectionManager] Setting role to datasource:kylin_learn role:ROLE_XS
2019-10-25 13:18:28,058 INFO  [org.saiku.web.core.SecurityAwareConnectionManager] Setting role to datasource:kylin_learn role:ROLE_XS
2019-10-25 13:18:28,059 INFO  [org.saiku.web.core.SecurityAwareConnectionManager] Setting role to datasource:kylin_learn role:ROLE_XS

```

确认role是否正确。

##### 说明

此种配置方法与官网介绍的配置方法不一致，可能是由于使用最新版本缘故。

此种配置方法是遵循LDAP协议完成，LDAP的介绍请百度。