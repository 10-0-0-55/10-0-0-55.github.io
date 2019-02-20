# SSRF相关

## SSRF之PHP
### filter_val parse_url 限定host bypass
利用条件：

* 使用filter_val
* 用parse_url限定host
* 限定host时使用了正则表达式

`filter_var($url, FILTER_VALIDATE_URL)`返回`$url`是否是一个有效的url
payload:

* `url=0://evil.com:80,felinae.cn:80/`
* `url=0://evil.com:23333;felinae98.cn:80/`

Reference：

* https://medium.com/secjuice/php-ssrf-techniques-9d422cb28d51
* https://skysec.top/2018/03/15/Some%20trick%20in%20ssrf%20and%20unserialize()/

### libcurl parse_url()
利用条件：

* 用parse_url限制host
* 限制了schema（不然可以使用上一个）

原理`http://u:p@a.com:80@b.com/`

parse_url解析结果：
```
schema: http 
host: b.com
user: u
pass: p@a.com:80
```
libcurl解析结果：
```
schema: http
host: a.com
user: u
pass: p
port: 80
```
`@b.com`会被忽略