# VUE前后端分离脚手架环境搭建

## 一、下载安装node.js
在node.js中文官网http://nodejs.cn/正常下载安装node.js就可以了。
安装完成后查看版本，是否安装成功。
```shell script
node -v
```

## 二、安装淘宝镜像
安装cnpm的原因：  
npm的服务器是外国的，所以有时候我们安装“模块”会很很慢很慢超级慢。
安装方法：打开命令行工具输入：
```shell script
npm install cnpm -g --registry=https://registry.npm.taobao.org
```

## 三、全局安装 vue-cli
在命令提示窗口执行：
```shell script
cnpm install -g vue-cli
```
安装完成后查看版本
```shell script
vue -V
```
## 四、生成项目文件
安装vue-cli成功后，通过cd命令进入你想放置项目的文件夹，在命令提示窗口执行创建vue-cli工程项目的命令：
```shell script
vue init webpack
```
## 五、生成文件目录后，使用 cnpm 安装依赖：
```shell script
cnpm install
```

##  六、启动
最后需要执行命令来启动项目
```shell script
npm run dev
```
访问浏览器，localhost:8080即可
