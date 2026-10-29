# Gomail

[![build](https://github.com/verystar/gomail/actions/workflows/build.yml/badge.svg)](https://github.com/verystar/gomail/actions/workflows/build.yml)
[![Go Reference](https://pkg.go.dev/badge/github.com/verystar/gomail.svg)](https://pkg.go.dev/github.com/verystar/gomail)
[![Go Report Card](https://goreportcard.com/badge/github.com/verystar/gomail)](https://goreportcard.com/report/github.com/verystar/gomail)
[![License](https://img.shields.io/badge/license-MIT-blue.svg)](LICENSE)

> 本项目 Fork 自 [go-gomail/gomail](https://github.com/go-gomail/gomail)，原项目已停止维护。
> 本仓库在保留原有 API 的基础上做了模块化与维护更新，import 路径为 `github.com/verystar/gomail`。

## Introduction

Gomail 是一个简单高效的邮件发送库，经过充分测试并有完善的文档。

Gomail 目前只支持通过 SMTP 服务器发送邮件，但 API 设计灵活，可以很容易地扩展为使用本地
Postfix、第三方 API 等方式发送。

本 Fork 相比上游的主要变化：

- 使用 Go Modules，module 路径为 `github.com/verystar/gomail`（不再使用 `gopkg.in/gomail.v2`）
- 移除 Go 1.4 及更早版本的兼容代码（`mime_go14.go`）
- 使用 GitHub Actions 替代 Travis CI 进行构建与测试
- 增加 `Makefile`，支持一键格式化检查、测试与覆盖率统计

要求 Go 1.17 或更高版本（CI 在 Go 1.19 / 1.20 上运行测试），无任何外部依赖。

## Features

- 附件（Attachments）
- 内嵌图片（Embedded images）
- HTML 与纯文本模板
- 特殊字符自动编码
- SSL 与 TLS
- 使用同一个 SMTP 连接发送多封邮件

## Documentation

- API 文档：<https://pkg.go.dev/github.com/verystar/gomail>

## Download

```bash
go get github.com/verystar/gomail
```

## Examples

### 发送一封带附件的邮件

```go
package main

import (
	"github.com/verystar/gomail"
)

func main() {
	m := gomail.NewMessage()
	m.SetHeader("From", "alex@example.com")
	m.SetHeader("To", "bob@example.com", "cora@example.com")
	m.SetAddressHeader("Cc", "dan@example.com", "Dan")
	m.SetHeader("Subject", "Hello!")
	m.SetBody("text/html", "Hello <b>Bob</b> and <i>Cora</i>!")
	m.Attach("/home/Alex/lolcat.jpg")

	d := gomail.NewDialer("smtp.example.com", 587, "user", "123456")

	// Send the email to Bob, Cora and Dan.
	if err := d.DialAndSend(m); err != nil {
		panic(err)
	}
}
```

### 复用 SMTP 连接批量发送（守护进程模式）

```go
package main

import (
	"log"
	"time"

	"github.com/verystar/gomail"
)

func main() {
	ch := make(chan *gomail.Message)

	go func() {
		d := gomail.NewDialer("smtp.example.com", 587, "user", "123456")

		var s gomail.SendCloser
		var err error
		open := false
		for {
			select {
			case m, ok := <-ch:
				if !ok {
					return
				}
				if !open {
					if s, err = d.Dial(); err != nil {
						panic(err)
					}
					open = true
				}
				if err := gomail.Send(s, m); err != nil {
					log.Print(err)
				}
			// Close the connection to the SMTP server if no email was sent in
			// the last 30 seconds.
			case <-time.After(30 * time.Second):
				if open {
					if err := s.Close(); err != nil {
						panic(err)
					}
					open = false
				}
			}
		}
	}()

	// Send messages to the channel.
	m := gomail.NewMessage()
	m.SetHeader("From", "alex@example.com")
	m.SetHeader("To", "bob@example.com")
	m.SetHeader("Subject", "Hello!")
	m.SetBody("text/plain", "Hello Bob!")
	ch <- m

	close(ch)
}
```

更多示例见 [example_test.go](example_test.go) 与
[文档中的示例](https://pkg.go.dev/github.com/verystar/gomail#pkg-examples)。

## Development

```bash
make fmt        # 格式化代码
make fmt-check  # 检查代码格式
make test       # 运行测试并生成 cover.out
make cover      # 生成覆盖率报告（cover_total.out / cover.html）
make vet        # go vet
make lint       # golangci-lint（需本地安装）
```

## FAQ

### x509: certificate signed by unknown authority

如果出现该错误，说明客户端认为 SMTP 服务器使用的证书无效。作为临时解决办法，可以通过
`SetTLSConfig` 跳过对服务器证书链和主机名的校验：

```go
package main

import (
	"crypto/tls"

	"github.com/verystar/gomail"
)

func main() {
	d := gomail.NewDialer("smtp.example.com", 587, "user", "123456")
	d.TLSConfig = &tls.Config{InsecureSkipVerify: true}

	// Send emails using d.
}
```

注意：该方式是不安全的，请勿在生产环境中使用。

## Contribute

欢迎提交 Issue 与 Pull Request，详见 [CONTRIBUTING.md](CONTRIBUTING.md)。

## Change log

见 [CHANGELOG.md](CHANGELOG.md)。

## License

[MIT](LICENSE)

## Contact

原项目讨论见 Go 邮件列表中的
[Gomail thread](https://groups.google.com/d/topic/golang-nuts/jMxZHzvvEVg/discussion)。
本 Fork 的问题请直接在 [Issues](https://github.com/verystar/gomail/issues) 中反馈。
