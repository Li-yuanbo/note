### curl常用命令

*curl* 是常用的命令行工具，用来请求 Web 服务器。名字的意思是：客户端*Client*的*URL*工具。

不带任何参数时，*curl* 就是发出 GET 请求：

```shell
$ curl https://www.baidu.com
```

#### -A

`-A `参数指定客户端的用户代理标头，即`User-Agent`，默认用户代理字符串是 `curl/[version]`。

将`User-Agent`指定为 Chrome 浏览器：

```shell
$ curl -A 'Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/76.0.3809.100 Safari/537.36' https://www.baidu.com
```

也可以通过 `-H` 参数直接指定标头，更改`User-Agent`：

```shell
$ curl -H 'User-Agent: php/1.0' https://www.baidu.com
```

#### -b

`-b` 参数用来向服务器发送 Cookie，向服务器发送一个 `key = "foo"`，`value = "bar"` 的 Cookie：

```shell
$ curl -b 'foo=bar' https://www.baidu.com
```

还可以读取本地文件 `cookies.txt`，将其发送到服务器：

```shell
$ curl -b cookies.txt https://www.baidu.com
```

#### -c

`-c` 参数将服务器设置的 Cookie 写入指定路径文件：

```shell
$ curl -c cookies.txt https://www.baidu.com
```

上述命令可以将服务器的 HTTP 回应所设置的 Cookie 写入文本文件 `cookies.txt`。

#### -d

`-d` 参数用于发送 POST 请求的数据体。使用 `-d` 参数后，HTTP 请求会自动加上标头`Content-Type : application/x-www-form-urlencoded`，并且自动将请求转换为 POST 方法，可以省略 `-X POST`。

```shell
$ curl -d'login=emma＆password=123'-X POST https://www.baidu.com/login
# 或者
$ curl -d 'login=emma' -d 'password=123' -X POST  https://www.baidu.com/login
```

#### --data-urlencode

`--data-urlencode`  参数等同于 `-d`，发送POST请求的数据体，区别在于会将发送的数据进行URL编码：

```shell
$ curl --data-urlencode 'comment=hello world' https://www.baidu.com/login
```

上述请求中，发送的数据中间有个空格，需要进行 URL 编码。

#### -e

`-e` 参数用来设置 HTTP 标头 `Referer` ，表示请求的来源，可以将标头设为`https://google.com?q=example`：

```shell
$ curl -e 'https://google.com?q=example' https://www.baidu.com
```

也可以通过 `-H` 参数直接添加标头 `Referer`，达到同样效果：

```shell
$ curl -H 'Referer: https://google.com?q=example' https://www.baidu.com
```

#### -F

`-F` 参数用来向服务器上传二进制文件，会默认给 HTTP 请求加上 `Content-Type: multipart/form-data`，然后将文件作为 *file* 字段上传：

```shell
$ curl -F 'file=@photo.png' https://www.baidu.com/profile
```

也可以指定文件类型和文件名称：

```shell
$ curl -F 'file=@photo.png;type=image/png;filename=server.png' https://www.baidu.com/profile
```

上述命令指定上传文件类型为 *image/png*，名称为 *server.png*。

#### -G

`-G` 参数用来构造 URL 的查询字符串：

```shell
$ curl -G -d 'q=query' -d 'dount=20' https://www.baidu.com/search
```

上述命令会发出一个 GET 请求，实际请求URL：`https://www.baidu.com/search?q=query&count=20`。

如果需要 URL 编码，可以结合 `--data--urlencode` 参数：

```shell
$ curl -G --data-urlencode 'q=query str' https://www.baidu.com/search
```

#### -H

`-H` 参数添加 HTTP 请求的标头：



























