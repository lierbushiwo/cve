# php-8.0.6 `php -S` 源码审计记录

## 审计对象

- 上游版本: `php-8.0.6`
- 提交: `ee186036499d62c33e68dae24c74cc36b7532faa`
- 关键文件:
  - `sapi/cli/php_http_parser.c`
  - `sapi/cli/php_cli_server.c`

## 结论

漏洞点不是单一函数，而是两段逻辑叠加造成的请求状态污染：

1. `php_http_parser_execute()` 在一次 `recv()` 的 buffer 中会继续解析后续请求。
2. `php_cli_server_client_read_request_on_message_begin()` 不重置 `client->request`。
3. `php_cli_server_request_translate_vpath()` 在“带点但不存在”的路径上直接 `return`，却不会清空旧的 `path_translated` / `path_info` / `sb`。
4. `php_cli_server_dispatch()` 仅依据 `ext=="php"` 且 `path_translated!=NULL` 判断是否走脚本分发。

这样会形成错配：

- `ext` 来自第二条请求 `/xx.php`
- `path_translated` 残留第一条请求 `/flag.txt`

最终导致静态文件按 PHP 脚本执行。

## 关键代码

### 1. parser 允许同一 buffer 继续进入下一条消息

- `sapi/cli/php_http_parser.c:252`
  - `# define NEW_MESSAGE() start_state`
- `sapi/cli/php_http_parser.c:1357-1366`
  - 请求无 body 时调用 `message_complete` 后继续 `state = NEW_MESSAGE()`

### 2. 新消息开始时未重置 request

- `sapi/cli/php_cli_server.c:1561-1563`
  - `php_cli_server_client_read_request_on_message_begin()` 直接 `return 0;`

### 3. 路径转换函数在不存在的“静态风格路径”上提前返回

- `sapi/cli/php_cli_server.c:1391-1394`
  - 路径中含 `.` 就设置 `is_static_file = 1`
- `sapi/cli/php_cli_server.c:1425-1430`
  - 找不到真实文件且 `is_static_file` 为真时直接 `return`
- 该 `return` 前没有清空旧的 `request->path_translated`

### 4. 脚本分发仅依赖后缀和旧路径是否非空

- `sapi/cli/php_cli_server.c:1717-1736`
  - `on_message_complete()` 重新根据当前 `vpath` 计算 `ext`
- `sapi/cli/php_cli_server.c:2172-2175`
  - `ext == php` 且 `path_translated != NULL` 时走脚本分发

## 根因判断

这是一个典型的跨请求状态复用问题，不是 PHP 解释器错误执行了静态文件，而是 CLI server 在同一连接/同一读取周期内把两条请求混进了同一个 `client->request` 对象。

## 修复方向

可选修复至少应满足其一：

1. 在 `on_message_begin()` 重置或重建 `client->request`
2. 在 `php_cli_server_request_translate_vpath()` 进入前先清空 `path_translated/path_info/ext/sb`
3. 在 `translate_vpath()` 早退前显式将 `path_translated = NULL`
4. 在 `php_cli_server_client_read_request()` 中读取到一条完整请求后立即停止继续解析后续字节，并为剩余字节引入独立缓冲区

## 当前最合理的漏洞描述

`php -S` 内置服务器在处理同一 TCP 连接中的连续请求时，请求对象状态未正确隔离；当第二个请求为不存在的 `.php` 路径时，会错误复用前一个静态文件请求的 `path_translated`，导致静态文件内容被作为 PHP 脚本执行。
