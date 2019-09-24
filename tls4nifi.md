#TLS方式配置NIFI权限
##1.使⽤用keytool⽣生成秘钥
生成keystore 

以下命令⽣生成⼀一个包含⾃自签证书（self-signed certificate）的Java keystore： 
```shell script
keytool -genkey -keyalg RSA -alias nifi -keystore keystore.jks -keypass [password] -
storepass [password] -validity 365 -keysize 4096 -dname "CN=xxx, O=xxx, OU=xx, C=xx"
```
其中，替换 [password] 为你想设置的密码，替换DN "CN=Admin User, O=Primeton, OU=NiFi,
C=CN" 。 

这样，我们就⽣生成了了 keystore.jks ⽂文件。该⽂文件需要放置到Nifi当中。 

##2. ⽣生成PKCS12⽂文件以及对应的Truststore
PKCS12 ⽂文件是⼀一种加密⽂文件，⼀一般⽤用于存放证书以及对应的私钥。由于使⽤用keytool⽆无法直接⽣ 
成 PKCS12 ⽂文件，我们⾸首先⽣生成⼀一个包含⾃自签证书的keystore（与上⽂文⽣生成Keystore的命令很相似）： 
```shell script
keytool -genkey -keyalg RSA -alias client -keystore client_keystore.jks -keypass
[password] -storepass [password] -validity 365 -keysize 4096 -dname "CN=xxx, O=xxx,
OU=xx, C=xx"
```
这⾥里里，我们只是随便便设置了了⼀一个密码 [password] ，因为这个Keystore只是⼀一个过渡的产物，我们最后不不
会⽤用到，所以随便便设置⼀一个就好。 

接着我们把这个keystore转化成PKCS12⽂文件：
```shell script
keytool -importkeystore -srckeystore client_keystore.jks -destkeystore client.p12 -
srcstoretype JKS -deststoretype PKCS12 -srcstorepass [password] -deststorepass
[client_password] -destkeypass [client_password] -srcalias client -destalias client
```

其中，替换 [xxpassword] 为你想为PKCS12⽂文件设置的密码。 
除了了⽣生成PKCS12⽂文件外，我们还需要⽣生成⼀一个信任PKCS12密匙⽂文件中的证书的truststore。为此，我们先从
之前的keystore中输出密匙的证书： 
```shell script
keytool -export -keystore client_keystore.jks -alias client -file client.der -storepass
[password]
```
当我们得到证书以后，我们把这个证书引⼊入到 truststore.jks 当中： 
```shell script
keytool -import -file client.der -alias client -keystore truststore.jks -storepass
[truststore_password] -noprompt
```

其中，替换 [truststore_password] 为你想为truststore设置的密码。 
这样，我们就⽣生成了了⼀一个要放到Nifi当中的 truststore.jks ⽂文件，以及需要导⼊入到浏览器器中的
client.p12 ⽂文件。 
最后，为了了安全起⻅见，你应该删除掉 client_keystore.jks 以及 client.der 两个⽂文件。 

##3. 导⼊入PKCS12⽂文件
想要通过TLS身份验证⽅方式访问Nifi UI，你必须先把⽣生成的PKCS12⽂文件导⼊入到浏览器器中。以Chrome为例例，先
进⼊入 设置 =>  ⾼高级 ，找到  证书管理理 ⼀一项 
点击导⼊入，按照向导指引，选择你的 client.p12 ⽂文件，并输⼊入该⽂文件的密码就可以完成导⼊入。然后你就
可以通过浏览器器访问加密的Nifi界⾯面了了。当被要求选择证书时，选择你添加的那项就可以了了 

##4. 修改nifi.properties配置文件 
位置在nifi安装目录的conf目录下：  
修改内容如下：  
```shell script
##site to site
nifi.remote.input.host=10.121.89.21 //主机IP
nifi.remote.input.secure=true
nifi.remote.input.socket.port=10000  //端口

## web
### 去掉8080
nifi.web.http.host=
nifi.web.http.port=
### 开启https 8443端口
nifi.web.https.host=10.121.89.21 //nifi主机IP或者localhost
nifi.web.https.port=8443
### 配置keystore ###
nifi.security.keystore=/perltmp/opt/certs/keystore.jks //keystore路径
nifi.security.keystoreType=JKS   //类型
nifi.security.keystorePasswd=123456  //生成时设置的密码
nifi.security.keyPasswd=123456   //生成时设置的密码
nifi.security.truststore=/perltmp/opt/certs/truststore.jks //truststore路径
nifi.security.truststoreType=JKS  //类型
nifi.security.truststorePasswd=123456  //生产时设置的密码
```

##5. 修改authorizers.xml
同样文件在nifi安装目录的conf下：  
需要配置以下内容：  
```shell script
##51行左右
  <userGroupProvider>
        <identifier>file-user-group-provider</identifier>
        <class>org.apache.nifi.authorization.FileUserGroupProvider</class>
        <property name="Users File">./conf/users.xml</property>
        <property name="Legacy Authorized Users File"></property>
        <property name="Initial User Identity 1">CN=Admin User, O=Primeton, OU=NiFi, C=CN</property>   //填写生成密钥时设置的dname
    </userGroupProvider>
##250行左右
<accessPolicyProvider>
        <identifier>file-access-policy-provider</identifier>
        <class>org.apache.nifi.authorization.FileAccessPolicyProvider</class>
        <property name="User Group Provider">file-user-group-provider</property>
        <property name="Authorizations File">./conf/authorizations.xml</property>
        <property name="Initial Admin Identity">CN=Admin User, O=Primeton, OU=NiFi, C=CN</property>  //填写生成密钥时设置的dname
        <property name="Legacy Authorized Users File"></property>
        <property name="Node Identity 1"></property>
        <property name="Node Group"></property>
    </accessPolicyProvider>
```
##6. 设置完成后，重启nifi服务
启动后nifi的安装目录下conf中有两个文件会发生变化：
1. users.xml 中将生成<用户信息>`<user identifier="59db3ed5-95c8-30a9-af3e-4e557fd03e21"/>`类似信息
2. authorizations.xml 中将生成如下类似信息<权限信息>：  
```shell script
 <policies>
        <policy identifier="f99bccd1-a30e-3e4a-98a2-dbc708edc67f" resource="/flow" action="R">
            <user identifier="59db3ed5-95c8-30a9-af3e-4e557fd03e21"/>
        </policy>
        <policy identifier="262b8ea0-d593-37ed-8185-e277633ff479" resource="/data/process-groups/60facd62-016d-1000-1472-08b82188be13" action="R">
            <user identifier="59db3ed5-95c8-30a9-af3e-4e557fd03e21"/>
        </policy>
        <policy identifier="c2f9dc13-3e34-33f5-a724-62750dfbc076" resource="/data/process-groups/60facd62-016d-1000-1472-08b82188be13" action="W">
            <user identifier="59db3ed5-95c8-30a9-af3e-4e557fd03e21"/>
        </policy>
        <policy identifier="3972ec1b-ea5c-3b9b-b0b2-5e58c93ef8bb" resource="/process-groups/60facd62-016d-1000-1472-08b82188be13" action="R">
            <user identifier="59db3ed5-95c8-30a9-af3e-4e557fd03e21"/>
        </policy>
        <policy identifier="71b05882-eb01-3afc-9f0c-67690a5ca05b" resource="/process-groups/60facd62-016d-1000-1472-08b82188be13" action="W">
            <user identifier="59db3ed5-95c8-30a9-af3e-4e557fd03e21"/>
        </policy>
        <policy identifier="b8775bd4-704a-34c6-987b-84f2daf7a515" resource="/restricted-components" action="W">
            <user identifier="59db3ed5-95c8-30a9-af3e-4e557fd03e21"/>
        </policy>
        <policy identifier="627410be-1717-35b4-a06f-e9362b89e0b7" resource="/tenants" action="R">
            <user identifier="59db3ed5-95c8-30a9-af3e-4e557fd03e21"/>
        </policy>
        <policy identifier="15e4e0bd-cb28-34fd-8587-f8d15162cba5" resource="/tenants" action="W">
            <user identifier="59db3ed5-95c8-30a9-af3e-4e557fd03e21"/>
        </policy>
        <policy identifier="ff96062a-fa99-36dc-9942-0f6442ae7212" resource="/policies" action="R">
            <user identifier="59db3ed5-95c8-30a9-af3e-4e557fd03e21"/>
        </policy>
        <policy identifier="ad99ea98-3af6-3561-ae27-5bf09e1d969d" resource="/policies" action="W">
            <user identifier="59db3ed5-95c8-30a9-af3e-4e557fd03e21"/>
        </policy>
        <policy identifier="2e1015cb-0fed-3005-8e0d-722311f21a03" resource="/controller" action="R">
            <user identifier="59db3ed5-95c8-30a9-af3e-4e557fd03e21"/>
        </policy>
        <policy identifier="c6322e6c-4cc1-3bcc-91b3-2ed2111674cf" resource="/controller" action="W">
            <user identifier="59db3ed5-95c8-30a9-af3e-4e557fd03e21"/>
        </policy>
    </policies>
```
以上两个文件可以删除，启动nifi时会根据配置文件自动生成以上2个文件。  
若启动后访问nifi网页时，提示无权限或无此用户，可查看以上两个文件中内容是否正常。
##注意： dname 切记要一样，包括空格