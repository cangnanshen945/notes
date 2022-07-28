# host概念
Host 请求头指明了请求将要发送到的服务器主机名和端口号。

如果没有包含端口号，会自动使用被请求服务的默认端口（比如 HTTPS URL 使用 443 端口，HTTP URL 使用 80 端口）。

所有 HTTP/1.1 请求报文中必须包含一个Host头字段。对于缺少Host头或者含有超过一个Host头的 HTTP/1.1 请求，可能会收到400（Bad Request）状态码

# HTTP host header attack
参考[HTTP Host 头攻击 -- 学习笔记](https://blog.csdn.net/angry_program/article/details/109034421)
[密码重置攻击](https://cloud.tencent.com/developer/article/1803712)
# 涉及到的配合Host的攻击方式
## XSS攻击
参考[浅谈XSS攻击的那些事（附常用绕过姿势）](https://zhuanlan.zhihu.com/p/26177815)
## XXE漏洞攻击
参考[一篇文章带你深入理解漏洞之 XXE 漏洞](https://xz.aliyun.com/t/3357)
## SSRF
参考[SSRF漏洞攻击原理及防御方案](https://zhuanlan.zhihu.com/p/91819069)
# charles 设置dns spoofing和map remote的区别
dns spoofing会在dns解析时使用更改后的ip地址代替域名，看似map remote修改请求的url也可以做到，但是前者是替换了域名解析的过程，没有实际修改请求，所以请求的header不会变，即host地址是还是该域名。但是后者是修改了请求，意味着header中的host被修改了。</br>
而现在很多服务是在同一个实体主机上的不同端口甚至不同路径来提供的，dns解析后的ip请求到了实体主机后，服务器根据host路由到不同的端口路径来提供服务。因此修改了header中的host后，可能会导致路由到其他服务从而导致服务无法成功。