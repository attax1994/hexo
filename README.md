# hexo

## new
```shell
$ hexo new [layout] <title>
```
新建一篇文章。如果没有设置 layout 的话，默认使用 _config.yml 中的 default_layout 参数代替。如果标题包含空格的话，请使用引号括起来。

## generate
```shell
$ hexo generate
$ hexo g
```
生成静态文件。

## server
```shell
$ hexo server
```
启动服务器。默认情况下，访问网址为： http://localhost:4000/。
|选项|描述|
-p, --port	重设端口
-s, --static	只使用静态文件
-l, --log	启动日记记录，使用覆盖记录格式