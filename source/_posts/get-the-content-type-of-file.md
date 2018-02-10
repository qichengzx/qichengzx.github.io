title: Go 检测文件内容类型
categories: golang
date: 2018-02-10 19:45:26
tags:  [go]
---

有时候需要检测文件的内容类型或 [MIME](https://developer.mozilla.org/zh-CN/docs/Web/HTTP/Basics_of_HTTP/MIME_types) 类型，为此，需要打开文件并读取前512个字节（因为DetectContentType()函数值使用前512个字节），所以不需要读取更多内容。这个函数会返回一个 MIME 类型，如 application/json 或 image/jpeg。


```Go
package main

import (
	"fmt"
	"net/http"
	"os"
)

func main() {
	fileArr := []string{"file/1.pdf", "file/2.jpg", "file/3.docx", "file/4.xml", "file/5.azw3", "file/6.zip", "file/7.torrent"}

	for _, fileName := range fileArr {

		f, err := os.Open(fileName)
		if err != nil {
			panic(err)
		}
		defer f.Close()

		contentType, err := getFileContentType(f)
		if err != nil {
			panic(err)
		}

		fmt.Print(fmt.Sprintf("File Name : %v, Content Type : %s\n", fileName, contentType))
	}
}

func getFileContentType(out *os.File) (string, error) {
	buffer := make([]byte, 512)

	_, err := out.Read(buffer)
	if err != nil {
		return "", err
	}

	contentType := http.DetectContentType(buffer)

	return contentType, nil
}

```