---
title: "nginx源码安装和clion远程调试"
categories: nginx
---

### 源码下载
git clone https://github.com/nginx/nginx.git
git checkout stable-1.14


### clion 配置
Build,Execution,Deployment > Deployment 
sftp选择远程机器，mapping配置映射到的远程目录
Build,Execution,Deployment > toolchain > remote host调到第一个



### 编译
./configure --prefix=/home/happen/nginx/bin --with-debug --with-cc-opt='-g -O0'
会提示说缺少依赖，安装依赖
apt install -y libpcre3 libpcre3-dev
修改生成的makefile，添加
remote:
    $(MAKE) && $(MAKE) install

### clion配置运行
Build,Execution,Deployment > custom build targets
新建一个make,toolchain使用remote host

运行： Makefile Application 
target: 选择刚刚上一把创建的target 
excutable选择 ./bin/sbin/nginx 
program arguments: -t
完成，运行,控制台输出
```
/home/happen/nginx/bin/sbin/nginx -t
nginx: the configuration file /home/happen/nginx/bin/conf/nginx.conf syntax is ok
nginx: configuration file /home/happen/nginx/bin/conf/nginx.conf test is successful
```
全局搜索到syntax is ok,修改成syntax is ok!!,再次运行可以看到输出改变