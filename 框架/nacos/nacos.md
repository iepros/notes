# nacos
## nacos下载安装
> 见官网 https://nacos.io 或 https://github.com/alibaba/nacos

## centos部署
> 见官网

## 设置开机启动



`vim /lib/systemd/system/nacos.service`

> 添加如下代码



`[Unit]`
`Description=nacos`
`After=network.target`
`[Service]`
`Type=forking`
`ExecStart=/usr/local/nacos/nacos/bin/startup.sh -m standalone`
`ExecReload=/usr/local/nacos/nacos/bin/shutdown.sh`
`ExecStop=/usr/local/nacos/nacos/bin/shutdown.sh`
`PrivateTmp=true`
`[Install]  
WantedBy=multi-user.target`

> 保存退出后，执行：

`systemctl daemon-reload`

`systemctl enable nacos.service`

`systemctl start nacos.service`

## 验证

> 浏览器登陆：http://127.0.0.0:8848/nacos，用户名：nacos，密码nacos

## 报错

> which: no javac in (/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/root/bin)
>
> ERROR: Please set the JAVA_HOME variable in your environment, We need java(x64)! jdk8 or later is better! !!

解决办法：

> 进入到nacos安装目录的bin目录下，修改startup.sh
>
> - vi startup.sh
> - 在配置JDK的地方添加一行JDK的配置：[ ! -e "$JAVA_HOME/bin/java" ] && JAVA_HOME=/usr/java/jdk1.8.0_251
> - 保存并退出即可解决