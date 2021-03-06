# 查看网页源码
```shell
$ curl www.sina.com
如果要把这个网页保存下来，可以使用`-o`参数，这就相当于使用wget命令了。
$ curl -o [文件名] www.sina.com
```

# 自动跳转
有的网址是自动跳转的。使用`-L`参数，curl就会跳转到新的网址。
```shell
$ curl -L www.sina.com
```
键入上面的命令，结果就自动跳转为www.sina.com.cn。

# 显示头信息
`-i`参数可以显示http response的头信息，连同网页代码一起。
`-I`参数则是只显示http response的头信息。
```shell
$ curl -i www.sina.com
```

# 显示通信过程
`-v`参数可以显示一次http通信的整个过程，包括端口连接和http request头信息。
```shell
$ curl -v www.sina.com
如果你觉得上面的信息还不够，那么下面的命令可以查看更详细的通信过程。
$ curl --trace output.txt www.sina.com
或者
$ curl --trace-ascii output.txt www.sina.com
运行后，请打开output.txt文件查看。
```

# 发送表单信息
发送表单信息有GET和POST两种方法。GET方法相对简单，只要把数据附在网址后面就行。
```shell
$ curl example.com/form.cgi?data=xxx
POST方法必须把数据和网址分开，curl就要用到--data参数。
$ curl -X POST --data "data=xxx" example.com/form.cgi
如果你的数据没有经过表单编码，还可以让curl为你编码，参数是`--data-urlencode`。
$ curl -X POST--data-urlencode "date=April 1" example.com/form.cgi
```

# HTTP动词
curl默认的HTTP动词是GET，使用`-X`参数可以支持其他动词。
```shell
$ curl -X POST www.example.com
$ curl -X DELETE www.example.com
```

# 文件上传
假定文件上传的表单是下面这样：
```shell
<form method="POST" enctype='multipart/form-data' action="upload.cgi">
　　　　<input type=file name=upload>
　　　　<input type=submit name=press value="OK">
　　</form>
```
你可以用curl这样上传文件：
```shell
$ curl --form upload=@localfilename --form press=OK [URL]
```

# Referer字段
有时你需要在http request头信息中，提供一个referer字段，表示你是从哪里跳转过来的。
```shell
$ curl --referer http://www.example.com http://www.example.com
```

# User Agent字段
这个字段是用来表示客户端的设备信息。服务器有时会根据这个字段，针对不同设备，返回不同格式的网页，比如手机版和桌面版。
iPhone4的User Agent是
```language
Mozilla/5.0 (iPhone; U; CPU iPhone OS 4_0 like Mac OS X; en-us) AppleWebKit/532.9 (KHTML, like Gecko) Version/4.0.5 Mobile/8A293 Safari/6531.22.7
```
curl可以这样模拟：
```shell
$ curl --user-agent "[User Agent]" [URL]
```

# cookie
使用`--cookie`参数，可以让curl发送cookie。
```shell
$ curl --cookie "name=xxx" www.example.com
```
至于具体的cookie的值，可以从http response头信息的`Set-Cookie`字段中得到。

`-c cookie-file`可以保存服务器返回的cookie到文件，`-b cookie-file`可以使用这个文件作为cookie信息，进行后续的请求。
```shell
$ curl -c cookies http://example.com
$ curl -b cookies http://example.com
```

# 增加头信息
有时需要在http request之中，自行增加一个头信息。`--header`参数就可以起到这个作用。
```shell
$ curl --header "Content-Type:application/json" http://example.com
```

# HTTP认证
有些网域需要HTTP认证，这时curl需要用到`--user`参数。
```shell
$ curl --user name:password example.com
```

