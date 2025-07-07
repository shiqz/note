## 简介
curl 是一个强大的命令行工具，用于传输数据，支持多种协议（如 HTTP、HTTPS、FTP 等）。它广泛用于测试 API、下载文件、上传数据等场景。

## 基本语法
```bash
curl [options] [URL...]
```

## 常用选项
### 指定请求方法
可以使用 `-X` 或者 `--request` 来指定请求方法
```bash
curl -X GET https://example.com
curl -X POST https://example.com
curl -X PUT https://example.com
curl -X DELETE https://example.com
```
### 发送数据
`-d` 或 `--data`: 发送 POST 数据
```bash
curl -d "name=John&age=30" https://example.com
```
`--data-raw`: 发送原始数据（不处理特殊字符）
```bash
curl --data-raw '{"name":"John"}' https://example.com
```
`--data-binary`: 发送二进制数据
```bash
curl --data-binary @file.txt https://example.com
```
`-F` 或 `--form`: 发送 multipart/form-data
```bash
curl -F "file=@photo.jpg" https://example.com/upload
```

### 设置请求头
`-H` 或 `--header`: 添加请求头
```bash
curl -H "Content-Type: application/json" -H "Authorization: Bearer token" https://example.com
```

### 输出控制
`-i` 或 `--include`: 包含响应头
```bash
curl -i https://example.com
```

`-I` 或 `--head`: 只获取响应头
```bash
curl -I https://example.com
```

`-o` 或 `--output`: 将输出保存到文件
```bash
curl -o output.html https://example.com
```

`-O` 或 `--remote-name`: 使用远程文件名保存
```bash
curl -O https://example.com/file.zip
```

`-s` 或 `--silent`: 静默模式（不显示进度和错误信息）
```bash
curl -s https://example.com
```

`-v` 或 `--verbose`: 显示详细过程（调试用）
```bash
curl -v https://example.com
```

### 请求认证
`-u` 或 `--user`: 基本认证
```bash
curl -u username:password https://example.com
```

`--oauth2-bearer`: OAuth 2.0 Bearer Token
```bash
curl --oauth2-bearer token https://example.com
```


### 设置代理
`-x` 或 `--proxy`: 使用代理
```bash
curl -x http://proxy.example.com:8080 https://example.com
```


### 其他
`-L` 或 `--location`: 跟随重定向
```bash
curl -L https://example.com
```

`--limit-rate`: 限制传输速率
```bash
curl --limit-rate 100k https://example.com/file.zip
```

`-k` 或 `--insecure`: 允许不安全的 SSL 连接
```bash
curl -k https://example.com
```

`--compressed`: 请求压缩响应
```bash
curl --compressed https://example.com
```
