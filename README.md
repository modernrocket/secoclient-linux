# secoclient-linux
## 解决secoclient-linux-64-7.0.2.26在Ubuntu等操作系统开机报错的问题

### 报错排查

**开机时遇到类似报错：**

```bash
Error found when loading /etc/profile: /etc/init.d/SecoClientPromoteService.sh:31:[[:not found
```

**问题原因：**secoclient安装时写入的开机启动脚本第31行，使用了dash不支持的语法，而基于Ubuntu的系统sudo执行时默认解释器是dash，导致报错。

```bash
if [[ $OS =~ "Ubuntu" ]] #ubuntu启动过程
```

### 解决思路

SecoClientPromoteService必须要在后台运行才能启动VPN，否则点击桌面程序没有反应。

华为的实现方式是安装程序在/etc/profile中写入脚本，每次开机或者连接ssh都会尝试拉起。你会看到由于这个脚本、登录ssh时会出现ps的打印。执行secoclient的卸载脚本时会同步删除/etc/profile中的这段脚本。

安装程序写入/etc/profile的相关代码如下。

```bash
#add serv lunch to /etc/profile
haveLunch=$(grep -n "#SecoClientPromoteServiceBegin" /etc/profile)
if [ -z "$haveLunch" ];then
    echo '#SecoClientPromoteServiceBegin'>>/etc/profile
    echo '#Warnning Do not add any content between Begin and End'>>/etc/profile
    echo 'ps -ef |grep SecoClientPromoteService |grep -v grep'>>/etc/profile
    echo 'if [ $? -ne 0 ]'>>/etc/profile
    echo 'then'>>/etc/profile
    echo 'sh /etc/init.d/SecoClientPromoteService.sh start'>>/etc/profile
    echo 'fi'>>/etc/profile
    echo '#SecoClientPromoteServiceEnd'>>/etc/profile
fi
```

同时，如果是Ubuntu系统，安装程序会尝试执行：

```bash
update-rc.d SecoClientPromoteService.sh defaults 20
```

但由于if判断出错，一般不会生效。

### 解决方案

因此，我们可以利用systemd彻底解决这个问题。

首先，删除/etc/profile中从#SecoClientPromoteServiceBegin到#SecoClientPromoteServiceEnd的全部内容。

你也可以参考/usr/local/SecoClient/uninstall.sh的方式：

```bash
##########delete the content from /etc/profile##############                                                                   
sbegin=$(grep -n "#SecoClientPromoteServiceBegin" /etc/profile|cut -d : -f 1)                                                  
send=$(grep -n "#SecoClientPromoteServiceEnd" /etc/profile|cut -d : -f 1)                                                      
if [ -n "$sbegin" ]&&[ -n "$send" ];then                                                                                       
    sdelc=${sbegin}","${send}"d"                                                                                               
    sed -i $sdelc /etc/profile                                                                                                 
fi     
```

然后，删除/etc/init.d/SecoClientPromoteService.sh，这是一个硬链接，源文件在/usr/local/SecoClient/promote/SecoClientPromoteService.sh

然后执行：

```bash
update-rc.d -f SecoClientPromoteService.sh remove
```

由于一般没有成功注册sysv，特地删除SecoClientPromoteService.sh和remove大概率没有实际作用。只是以防万一。

最后，在/etc/systemd/system下，创建`secoclient-promote.service`文件，service名称也可以随意。

内容如下：

```ini
[Unit]
Description=SecoClientPromoteService daemon
After=network.target
Wants=network.target

[Service]
Type=forking
ExecStart=/usr/local/SecoClient/promote/SecoClientPromoteService -d
ExecStop=/bin/kill -9 $MAINPID
WorkingDirectory=/usr/local/SecoClient/promote
User=root
Restart=on-failure
RestartSec=2

[Install]
WantedBy=multi-user.target
```

然后执行：

```bash
sudo systemctl daemon-reload  #重新加载
sudo systemctl enable secoclient-promote.service  #开机自启动
sudo systemctl start secoclient-promote.service  #立刻启动
```

可以查看状态是不是active

```bash
sudo systemctl status secoclient-promote.service
```

最终，我们通过systemd代替了原来的启动脚本。由于运行VPN必须要有这个后台进程，可以用systemd让他开机自启动，或者，你也可以每次使用VPN手动使用start和stop，也可以正常启动和停止。

如果需要卸载，卸载时还需要自行关闭、删除这个service。
