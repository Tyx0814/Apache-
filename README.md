Web服务器有三大主流，分别是Apache(httpd)，Nginx，IIS，这个是关于Apache解析漏洞
Apache的多后缀解析规则缺陷：Apache解析⽂件时，会从右向左识别⽂件后缀，直到识别到⾃⼰能解析的合法后缀为⽌，忽略中间不认识的后缀。
举例：Apache默认能解析 .php 后缀，当访问 test.php.abc 时，Apache不识别 .abc ，向左识别到 .php ，会将该⽂件当作PHP脚本执⾏；
补充：若配置了 AddHandler php5-script .php ，则所有包含 .php 后缀的⽂件都会被解析为PHP
所以攻击者可上传 xxx.php.xxx 这类畸形后缀⽂件，绕过⽹站的⽂件上传后缀⿊名单限制，上传的恶意PHP脚本被Apache解析执⾏，可直接获取服务器权限，写⼊后⻔、窃取数据、控制 服务器。
这次的靶场是vulhub，可以选择ubuntu_vulhub靶机。
OK 最后分享我担的一句话 “对我来说，风光无限的是你，跌落尘埃的也是你，重要的是你，而不是怎样的你”
