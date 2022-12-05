---
title: "Golang: capturing logs and send an email"
date: 2019-06-30
tags: ["go", "email", "logs"]
---

In my company we have an ETL wrote in Golang to process the integrations with our partners, each integration is executed in an unique and isolate POD using cronjob k8s, each one print a bunch of data and metrics for each step executed using `log` the package in the standard library, all these logs are useful to monitor the integrations with different tools.

In my team now we want to receive an email when some integration is failed with the logs of the process, so for that, we use a feature of `log` to change the output destination for the standard logger called ` SetOutput`.

Using `io.MultiWriter` we can create a writer combine multiple writers, in this case, we will combine a buffer with the standard `os.Stderr` to save the logs into the buffer and keep logs in `stderr` for monitorization.

*The package `log` use `os.Stderr` for default.*


```golang
		buf := new(bytes.Buffer)
		w := io.MultiWriter(buf, os.Stderr)
		log.SetOutput(w)
```

Then, all our logs are into the buffer, so we can create a file to save it in a temporal file and attach it to an email.

```golang
	f, err := os.Create("/path/log.txt")
	if err != nil {
		// Handler ....
	}

	defer f.Close()

	// We copy the buffer into the file.
	if _, err := io.Copy(f, buf); err != nil {
		// Handler ....
	}
```


Now we can send an email with the logs file, I found some implementations to send an email with attachment file using only the standard libraries but they didn't work, so I combine different approach and got this implementation.


```golang
package main

import (
	"bytes"
	"encoding/base64"
	"fmt"
	"io/ioutil"
	"mime/multipart"
	"net/smtp"
	"os"
	"path/filepath"
)

var (
	host       = os.Getenv("EMAIL_HOST")
	username   = os.Getenv("EMAiL_USERNAME")
	password   = os.Getenv("EMAIL_PASSWORD")
	portNumber = os.Getenv("EMAIL_PORT")
)

type Sender struct {
	auth smtp.Auth
}

type Message struct {
	To          []string
	Subject     string
	Body        string
	Attachments map[string][]byte
}

func New() *Sender {
	auth := smtp.PlainAuth("", username, password, host)
	return &Sender{auth}, nil
}

func (s *Sender) Send(m *Message) error {
	return smtp.SendMail(fmt.Sprintf("%s:%s", host, portNumber), s.auth, username, m.To, m.ToBytes())
}

func NewMessage(s, b string) *Message {
	return &Message{Subject: s, Body: b, Attachments: make(map[string][]byte)}
}

func (m *Message) AttachFile(src string) error {
	b, err := ioutil.ReadFile(src)
	if err != nil {
		return err
	}

	_, fileName := filepath.Split(src)
	m.Attachments[fileName] = b
	return nil
}

func (m *Message) ToBytes() []byte {
	buf := bytes.NewBuffer(nil)
	withAttachments := len(m.Attachments) > 0

	buf.WriteString(fmt.Sprintf("Subject: %s\n", m.Subject))
	buf.WriteString("MIME-Version: 1.0\n")
	writer := multipart.NewWriter(buf)
	boundary := writer.Boundary()

	if withAttachments {
		buf.WriteString(fmt.Sprintf("Content-Type: multipart/mixed; boundary=%s\n", boundary))
		buf.WriteString(fmt.Sprintf("--%s\n", boundary))
	}

	buf.WriteString("Content-Type: text/plain; charset=utf-8\n")
	buf.WriteString(m.Body)

	if withAttachments {
		for k, v := range m.Attachments {
			buf.WriteString(fmt.Sprintf("\n\n--%s\n", boundary))
			buf.WriteString("Content-Type: application/octet-stream\n")
			buf.WriteString("Content-Transfer-Encoding: base64\n")
			buf.WriteString(fmt.Sprintf("Content-Disposition: attachment; filename=%s\n", k))

			b := make([]byte, base64.StdEncoding.EncodedLen(len(v)))
			base64.StdEncoding.Encode(b, v)
			buf.Write(b)
			buf.WriteString(fmt.Sprintf("\n--%s", boundary))
		}

		buf.WriteString("--")
	}

	return buf.Bytes()
}

func main() {
	sender := New()
	m := NewMessage("Test", "Body message.")
	m.To = []string{"to@gmail.com"}
	m.AttachFile("/path/to/file")
	fmt.Println(s.Send(m))
}
```

***If you have some tips to improve this implementation or one way to do better that would be amazing ...***

[Github](https://gist.github.com/douglasmakey/90753ecf37ac10c25873825097f46300).