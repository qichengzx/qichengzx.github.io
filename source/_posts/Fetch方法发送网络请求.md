layout: react-native
title: Fetch方法发送网络请求
date: 2015-11-06 22:54:22
tags:  [react-native,fetch,网络请求]
---



先贴一个官方文档。
[Network](https://facebook.github.io/react-native/docs/network.html#content)



    fetch('https://mywebsite.com/endpoint/', {
        method: 'POST',
        headers: {
            'Accept': 'application/json',
            'Content-Type': 'application/json',
        },
        body: JSON.stringify({
            firstParam: 'yourValue',
            secondParam: 'yourOtherValue',
        })
    })

一开始我就是单纯的这么写的，愉快的把URL替换掉，把参数替换掉，然后。。。
我愉快的敲下 `var_dump($_POST['firstParam'])` 的时候，后台没接到任何东西，后来在stackoverflow上看到回答，说之所以接收不到，是因为这不属于form data，想要这么写，需要在接收前添加 `file_get_contents('php://input') `,然后果然就可以了。

    $json = json_decode(file_get_contents('php://input'), true);
    var_dump($json['firstParam']);

但是这样会影响到现有的web应用，所以又继续查资料，发现可以改成下边这样的方法。


    function toQueryString(obj) {
        return obj ? Object.keys(obj).sort().map(function (key) {
            var val = obj[key];
            if (Array.isArray(val)) {
                return val.sort().map(function (val2) {
                    return encodeURIComponent(key) + '=' + encodeURIComponent(val2);
                }).join('&');
            }
    
            return encodeURIComponent(key) + '=' + encodeURIComponent(val);
        }).join('&') : '';
	}


    fetch(url, {
        method: 'post',
        body: toQueryString({ 
            'firstParam': 'yourValue',
            'secondParam':'yourOtherValue' 
        })
    }) 



这样，在后台就可以正常的用$_POST['firstParam'];接收了。


[Sending data as key-value pair using fetch polyfill in react-native](http://stackoverflow.com/questions/32448862/how-can-i-use-react-native-with-php-with-fetch-return-data-is-always-null)

[How can I use react-native with php with fetch return data is always null](http://stackoverflow.com/questions/31201940/sending-data-as-key-value-pair-using-fetch-polyfill-in-react-native)


