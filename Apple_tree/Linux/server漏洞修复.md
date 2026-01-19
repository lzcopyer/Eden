## 隐藏nginx server信息

停止当前的nginx，进入解压出来的nginx 源码目录（不是nginx的安装目录）重新编译，但不安装

``` bash
cd {{ nginx_bak }}
#检查nginx安装信息
sbin/nginx -t
vim src/http/ngx_http_header_filter_module.c # 49-50行 删除nginx信息
vim src/http/ngx_http_special_response.c #36行
vim src/core/nginx.h
./configure
make

#nginx 热升级
cd {{ nginx_home }}
mv nginx nginx_bak
cp {{ nginx_bak/objs/nginx }} ./
make upgrade
```

## tomcat 默认文件漏洞

``` bash
vim {{ tom_dir }}/lib/catalina.jar #选择ServerInfo.properties 删除一下关于tomcat版本信息
server.info=Apache Tomcat/8.5.23
server.number=8.5.23.0
server.built=Sep 28 2017 10:30:11 UTC

rm -rf {{ tom_dir }}/webapps/{docs,examples}

vim {{ tom_dir }}/conf/web.xml #添加下面内容
<error-page>
<error-code>400</error-code>
<location>/error.html</location>
</error-page>
<error-page>
<error-code>404</error-code>
<location>/error.html</location>
</error-page>
<error-page>
<error-code>500</error-code>
<location>/error.html</location>
</error-page>

vim {{ tom_dir }}/webapps/ROOT/error.html
<!DOCTYPE html>
<html xmlns="http://www.w3.org/1999/xhtml"><head>
<title>网页访问不了</title>
<meta http-equiv="Content-Type" content="text/html; charset=utf-8">
<link rel="stylesheet" type="text/css" href="404/error_all.css?t=201303212934">
</head>
<body class="error-404">
<div id="doc_main">
<section class="bd clearfix">
<div class="module-error">
<div class="error-main clearfix">
<div class="label"></div>
<div class="info">
<h3 class="title">Sorry，你所访问的页面有问题哦</h3>
<div class="reason">
<p>可能的原因：</p >
<p>1.手写有问题。</p >
<p>2.URL失效了？</p >
</div>
</div>
</div>
</div>
</section>
</div>
</body>
</html>
```
