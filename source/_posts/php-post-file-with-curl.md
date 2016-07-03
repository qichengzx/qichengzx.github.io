title: PHP Curl 上传文件
categories: php
date: 2016-07-02 23:11:30
tags:  [php]

---

有时候会遇到上传文件给第三方服务的情况，比如本身程序并不需要存储附件，而是把附件发送给一个公共的服务。

最近正好碰到这个问题，记录一下。

上代码。

#### 发送端：

```php
<?php

// 接口地址
$api = 'http://api.example.com/uploadfile';
$file = $_FILES['file'];//保存$_FILES到变量中。

// 此处可能存在上传失败等问题，需验证$_FILES["file"]["error"]。
// 做业务对应的规则验证，如文件格式，文件大小等。

// 创建一个 cURL 句柄
$ch = curl_init($api);

// 创建一个 CURLFile 对象
// 上传文件的路径，文件的Mimetype，文件名
$cfile = curl_file_create($file['tmp_name'],$file['type'],$file['name']);

$data = [
	'type'=>'image',
	'data'=>$cfile
];

// 设置 POST 数据
curl_setopt($ch, CURLOPT_POST,1);
curl_setopt($ch, CURLOPT_POSTFIELDS, $data);

$resp = curl_exec($ch);

if(!$resp) {
  die('Error: "' . curl_error($ch) . '" - Code: ' . curl_errno($ch));
} else {
  echo "Response HTTP Status Code : " . curl_getinfo($ch, CURLINFO_HTTP_CODE);
  echo "\nResponse HTTP Body : " . $resp;
}

// Close request to clear up some resources
curl_close($ch);
```

#### 接收端：

```php
<?php 
var_dump($_FILES);// 接收文件内容
va_dump($_POST);// 接收type
```

#### 前端：

前端可使用Form或其他Ajax方式上传。