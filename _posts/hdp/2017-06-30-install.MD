## Vagrant Ubuntu14.4搭建Hadoop环境
需要在windows下安装三个软件:
 - [VirtualBox](https://www.virtualbox.org) 虚拟化  
 - [Vagrant](https://www.vagrantup.com/) 虚拟化管理  

下载ambari官方提供的vagrant配置文件[https://github.com/u39kun/ambari-vagrant.git](https://github.com/u39kun/ambari-vagrant.git)，[参考](https://cwiki.apache.org/confluence/display/AMBARI/Quick+Start+Guide)<br>
打开Windows CMD窗口，输入以下命令：
```
$ cd D:/05hadoop/01ambari/ambari-vagrant-master/ubuntu14.4   (进入ubuntu14.4系统的vagrant目录)
$ vagrant up u1401 u1402 u1403  (启动三个虚拟机，u1401:ambari server、u1402: hadoop Namenode、u1403: hadoop datanode)
```
启动三个虚拟机后，SSH工具登录，并分别设置允许ROOT用户ssh登录：
```
$ vi /etc/ssh/sshd_config
# Authentication:
LoginGraceTime 120
PermitRootLogin yes  #此处设置为yes
StrictModes yes
$ service restart ssh  #重启ssh服务
```

## ubuntu14下的环境准备
将3台VM的ubuntu官方源替换为163源，在3台VM上执行以下命令：
```
$ sed -i "s/us.archive.ubuntu.com/mirrors.163.com/" /etc/apt/sources.list
$ sed -i "s/security.ubuntu.com/mirrors.163.com/" /etc/apt/sources.list
$ apt-get update       (如果直接使用ubuntu官方源更新，会提示有hashsum错误)
```

计划在u1401上安装apache ambari server，在另两台VM上部署hadoop集群。所有三个VM都要添加ambari和hadoop的本地源：  
```
$ sudo su - root               (切换为root用户，提示符变成#号)
$ cd /etc/apt/sources.list.d && \
  wget http://repo.imaicloud.com/AMBARI-2.4.2.0/ubuntu14/2.4.2.0-136/ambari.list && \
  wget http://repo.imaicloud.com/hdp/HDP/ubuntu14/HDP.list  && \
  wget http://repo.imaicloud.com/hdp/HDP-UTILS-1.1.0.21/repos/ubuntu14/HDP-UTILS.list && \
 apt-key adv --recv-keys --keyserver keyserver.ubuntu.com B9733A7A07513CAD && \
 apt-get update
```
(注1：repo.imaicloud.com(10.0.9.105)是浪潮内网的本地源。使用本地源可以大大节省安装时间，并减少安装出错的概率。）  
(注2:```&&```符号表示顺序运行多个linux命令，反斜杠用于把很长的linux命令分成多行）  

u1402和u1403都添加本地源，并更新。

## 让u1401免密码登录其他VM
参考[SSH的入门](https://github.com/wbwangk/wbwangk.github.io/wiki/SSH%E5%85%A5%E9%97%A8)。  
ambari server要安装在u1401上，必须让u1401的root用户可以免密码登录u1402和u1403。  
在u1402和u1403（2号和3号窗口）上为root用户创建口令和修改ssh配置(改变ssh的默认设置PermitRootLogin为yes)：
```
$ passwd root                 （会提示输入密码）
```
在u1401上生成密钥对，用到openssh的命令：
```
$ ssh-keygen    (生成密钥对，回车3次)
$ cd ~/.ssh
$ ls -l
-rw------- 1 root root  409 Mar 23 10:18 authorized_keys
-rw------- 1 root root 1675 Mar 23 11:04 id_rsa
-rw-r--r-- 1 root root  392 Mar 23 11:04 id_rsa.pub
```
id_rsa是私钥，id_rsa.pub是公钥。  
复制公钥到u1402和u1403，在u1401上执行：
```
$ ssh-copy-id u1402    （输入yes并输入u1402的root密码，下同）
$ ssh-copy-id u1403
$ ssh u1402   (尝试一下免密码登录u1402，输入exit返回)
```
## u1401上安装ambari-server
在u1401上执行：
```
$ apt-get install ambari-server -y  && \
  ambari-server setup -s         (会自动安装jdk1.8，173M，较耗时)
$ ambari-server start
$ curl http://u1401:8080         (启动需要几分钟，用户名口令是admin/admin)
$ ping u1401                     (可以看到u1401的ip是192.168.14.101)
```
在windows下的浏览器中输入```http://192.168.14.101:8080```就可以调出ambari的界面，用```admin/admin```登录。  

## 利用ambari安装hadoop

用浏览器打开```http://192.168.14.101:8080```，登录密码admin/admin。
 1. 点击页面上“Launch Install Winzard”按钮  
 2. 命名集群，如输入dev_u14。  
 3. 选择版本，选择最新的HDP-2.5。选择Local Repository，删除其他库，仅留下ubuntu14，在Base URL处分别输入：
```
http://repo.imaicloud.com/hdp/HDP/ubuntu14
http://repo.imaicloud.com/hdp/HDP-UTILS-1.1.0.21/repos/ubuntu14
```
 4. 安装选项，在目标主机中输入：
```
u1402.ambari.apache.org
u1403.ambari.apache.org
```
还需要提供SSH Private Key。这个key存在于~/.ssh/目录下，可以这样查看：
```
$ cat ~/.ssh/id_rsa          （~代表了```/root```，即root用户的HOME目录）
```
把cat命令输出文件内容到屏幕，鼠标选择内容复制，粘贴到浏览器窗口的ssh private key输入框中。  
 5. 确认主机，问题点可能是免密码SSH或agent未启动。出问题就点击红框查看出错日志，逐个解决。  
 6. 选择服务，初学者可以选择HDFS、ZooKeeper、Ambari Metrics这三个服务。如果只选一个ZooKeeper会安装失败，原因未知。    
 7. 分配主机(master)，可以调整每个主机安装的服务，默认即可。  
 8. 分配从机(slave)和客户端，默认即可。  
 9. 配置服务，有红色提示，设定Grafana管理员密码，其他默认。  
 10. 审核，点击部署按钮。  
 11. 安装、启动和测试，显示部署的进度框。这一步耗时较长，也容易出错。需要自己看日志解决。  
 12. 总结，点完成按钮
浏览器切换到了面板页面。在服务页面上，点击Quick Links可以进入各个服务的界面（如果有的话）。  
参考[HDFS的官方文档]进行HDFS的测试：
```
$ curl http://192.168.14.102:50070/webhdfs/v1/?op=LISTSTATUS
{"FileStatuses":{"FileStatus":[
{"accessTime":0,"blockSize":0,"childrenNum":1,"fileId":16386,"group":"hdfs","length":0,"modificationTime":1490698439789,"owner":"hdfs","pathSuffix":"tmp","permission":"777","replication":0,"storagePolicy":0,"type":"DIRECTORY"},
{"accessTime":0,"blockSize":0,"childrenNum":1,"fileId":16387,"group":"hdfs","length":0,"modificationTime":1490698426134,"owner":"hdfs","pathSuffix":"user","permission":"755","replication":0,"storagePolicy":0,"type":"DIRECTORY"}
]}}
```
上述响应中tmp和user是HDFS中的两个目录。查询tmp目录下的内容：
```
curl http://192.168.14.102:50070/webhdfs/v1/tmp?op=LISTSTATUS
```
在u1402或u1403主机上使用命令行测试HDFS：
```
$ hdfs dfs -ls /   (反斜杠表示显示根目录下的文件清单)
Found 2 items
drwxrwxrwx   - hdfs   hdfs            0 2017-03-28 15:57 /tmp
drwxr-xr-x   - hdfs   hdfs            0 2017-03-28 18:05 /user
```
还可以用浏览器访问这个地址来显示HDFS（估计这个UI是HDP实现的）的文件清单：
```
http://192.168.14.102:50070/explorer.html#/
```
  
