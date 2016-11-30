title: Create a Single Page App With Go, Echo and Vue
date: 2016-11-30 17:28:55
tags:  [go,vue]
---

本教程中我们将会创建一个"todo"应用。完成后可以实现创建任务，展示最新创建的任务和删除它们。

此程序后端使用Go语言。Go由Google开发。虽然不是最流行的语言，但是正在逐步得到认可。Go非常轻量级，易于学习，运行快。此教程假设你已经对于这门语言有了一些了解，并且已经安装和配置好了开发环境。

我们将会使用Echo框架，Echo框架相当于PHP语言的 Slim PHP 或 Lumen 框架，你应该有点熟悉使用微框架和使用路由处理http请求的概念。

任务数据将会储存在SQLite数据库中，SQLite是一个轻量级的可替代MySQL或PostgreSQL的数据库。数据会存在一个独立的文件中，与应用在同一目录而不是存在服务器上。

最后，前端使用HTML5和流行的 VueJS JavaScript 框架，需要对 VueJS 有一定的了解。

应用会被分成4个基本功能。首先







## 原文链接:[create-a-single-page-app-with-go-echo-and-vue](https://scotch.io/tutorials/create-a-single-page-app-with-go-echo-and-vue)