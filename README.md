Web服务器有三大主流，分别是Apache(httpd)，Nginx，IIS，这个是关于Apache解析漏洞，
攻击者可上传 xxx.php.xxx 这类畸形后缀⽂件，绕过⽹站的⽂件上传后缀⿊名单限制，上传的恶意PHP脚本被Apache解析执⾏，可直接获取服务器权限，写⼊后⻔、窃取数据、控制 服务器。
这次的靶场是vulhub，可以选择ubuntu_vulhub靶机。
