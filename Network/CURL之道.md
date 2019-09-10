#查看网页源码
```shell
$ curl www.sina.com
如果要把这个网页保存下来，可以使用`-o`参数，这就相当于使用wget命令了。
$ curl -o [文件名] www.sina.com
```

#自动跳转
有的网址是自动跳转的。使用`-L`参数，curl就会跳转到新的网址。
```shell
$ curl -L www.sina.com
```
键入上面的命令，结果就自动跳转为www.sina.com.cn。

#显示头信息
`-i`参数可以显示http response的头信息，连同网页代码一起。
`-I`参数则是只显示http response的头信息。
```shell
$ curl -i www.sina.com
```

#显示通信过程
`-v`参数可以显示一次http通信的整个过程，包括端口连接和http request头信息。
```shell
$ curl -v www.sina.com
如果你觉得上面的信息还不够，那么下面的命令可以查看更详细的通信过程。
$ curl --trace output.txt www.sina.com
或者
$ curl --trace-ascii output.txt www.sina.com
运行后，请打开output.txt文件查看。
```

#发送表单信息
发送表单信息有GET和POST两种方法。GET方法相对简单，只要把数据附在网址后面就行。
```shell
$ curl example.com/form.cgi?data=xxx
POST方法必须把数据和网址分开，curl就要用到--data参数。
$ curl -X POST --data "data=xxx" example.com/form.cgi
如果你的数据没有经过表单编码，还可以让curl为你编码，参数是`--data-urlencode`。
$ curl -X POST--data-urlencode "date=April 1" example.com/form.cgi
```

#HTTP动词
curl默认的HTTP动词是GET，使用`-X`参数可以支持其他动词。
```shell
$ curl -X POST www.example.com
$ curl -X DELETE www.example.com
```

#
