# HTTP是什么？
HTTP（超文本传输协议）
是负责在两个设备直接传输超文本数据的协议
## 常见状态码
2xx 成功
3xx 重定向
4xx的代表客户端异常
401 登录验证被拒
403 权限不足
404 找不到文件资源
500 服务器笼统异常
502 服务器是正常的，但是代理转发出错
503 服务器在忙

## HTTP常见字段
Host主机,content-length报文长度,connection(keep-alive)默认长连接，content-type内容格式
content-encoding压缩方式

## GET和HOST的区别
Get一般不带请求体，post带请求体，其实本质上GET和POST没有区别。
但是一般语境下GET是幂等安全的，post不是