title: 【译】使用Go，Angular，WebSockets构建实时聊天应用
categories: golang
date: 2017-07-10 21:59:45
tags:  [go]

---

### 写在前面

本文[原文](https://www.thepolyglotdeveloper.com/2016/12/create-real-time-chat-app-golang-angular-2-websockets/)

#### 正文

我最近听到很多关于WebSocket的东西，以及WebSocket如何在应用程序和服务器之间实现实时通信。WebSocket作为RESTful API的替代和补充，已经存在了很长时间。使用WebSocket可以做例如实时聊天，与IoT通信，游戏，和其他很多需要在客户端和服务器之间进行即时消息传递的东西。

最近一段时间，我[使用了一个叫Socket.io的库，用来在Node.js中使用websockets](https://www.thepolyglotdeveloper.com/2016/01/create-a-real-time-chat-application-with-the-cean-stack-and-socket-io/)，但是当我真正使用Go以后，我打算研究一下如何在Go中使用WebSocket。

通过本文，我们将学习如何创建一个聊天应用，其中客户端是一个 Angular 2 应用，服务端使用Go。

#### 要求

在这个应用中有很多操作，所以有一些必要的前提条件，如下所示：

- [Go](https://golang.org/) 1.7+
- [Node.js](https://nodejs.org/en/) 4.0+
- [Angular2 CLI](https://cli.angular.io/)

处理所有消息和客户端的聊天服务器使用Go编写。客户端前端使用 Angular 2编写，has a dependency of the Node Package Manager (NPM) which ships with Node.js.

#### 创建Go聊天服务器

我们打算先开发整个应用的服务器端部分，它需要依赖几个第三方的包。

在命令行执行以下命令，下载第三方包：

```go
go get github.com/gorilla/websocket
go get github.com/satori/go.uuid
```

websocket包的作者同时也是 [Mux](https://github.com/gorilla/mux) 这个路由包 的作者，我们还需要一个UUID包来分配每一个客户端的唯一ID。

在 $GOPATH 目录创建一个新的项目，我自己的项目目录是 $GOPATH/src/github.com/nraboy/realtime-chat/main.go。

在进行下一步之前，需要注意的是，我从 [Dinosaurs Code](https://dinosaurscode.xyz/go/2016/07/17/go-websockets-tutorial/) 和 [Gorilla websocket chat example](https://github.com/gorilla/websocket/tree/master/examples/chat)  获取了一部分Go 代码，为了避免剽窃的嫌疑，我使用了很多原始代码中的一部分，但我也为这个项目加入了很多自己的独特的东西。

这次我们要做的聊天应用有3个结构体：

```go
// $GOPATH/src/github.com/nraboy/realtime-chat/main.go
type ClientManager struct {
    clients    map[*Client]bool
    broadcast  chan []byte
    register   chan *Client
    unregister chan *Client
}
 
type Client struct {
    id     string
    socket *websocket.Conn
    send   chan []byte
}
 
type Message struct {
    Sender    string `json:"sender,omitempty"`
    Recipient string `json:"recipient,omitempty"`
    Content   string `json:"content,omitempty"`
}
```

ClientManager用于管理所有已连接的客户端，尝试连接的客户端，已经断开连接等待删除的客户端，和所有已连接客户端收发的消息。

每个客户端有一个唯一的ID，一个socket连接，和等待发送的消息。

为了增加传递的数据的复杂性，消息将使用 JSON 格式。而不是传递一串不容易被理解，阅读的数据。使用JSON格式，我们可以使用元数据和其他有用的东西。每一条消息将包含发送消息的客户端，接收消息的客户端，和消息的实际内容。

首先定义一个全局的ClientManager。

```go
var manager = ClientManager{
    broadcast:  make(chan []byte),
    register:   make(chan *Client),
    unregister: make(chan *Client),
    clients:    make(map[*Client]bool),
}
```

服务器端将使用3个goroutine，一个用于管理客户端，一个用于读取websocket数据，另一个用于往websocket里写数据。这里指的是读取和写入的goroutine将为每个连接的客户端创建一个新的实例。所有的goroutine将循环运行直至不再需要。

编写如下代码，来开始服务：

```go
func (manager *ClientManager) start() {
    for {
        select {
        case conn := <-manager.register:
        	manager.clients[conn] = true
        	jsonMessage, _ := json.Marshal(&Message{Content: "/A new socket has connected."})
            manager.send(jsonMessage, conn)
        case conn := <-manager.unregister:
            if _, ok := manager.clients[conn]; ok {
                close(conn.send)
                delete(manager.clients, conn)
                jsonMessage, _ := json.Marshal(&Message{Content: "/A socket has disconnected."})
                manager.send(jsonMessage, conn)
            }
        case message := <-manager.broadcast:
            for conn := range manager.clients {
                select {
                case conn.send <- message:
                default:
                    close(conn.send)
                    delete(manager.clients, conn)
                }
            }
        }
    }
}
```

每当manager.register通道收到数据，这个客户端将会被添加到manager(前文创建的ClientManager实例)的register客户端中。然后，将向所有其他客户端发送一条JSON消息。

同时，如果客户端断开连接，manager.unregister通道将会收到消息，断开连接的客户端的通道中的数据将被关闭，客户端也会从manager中删除。然后发送消息给其他的客户端告知某个客户端已断开连接。

如果 manager.broadcast 通道中有数据，则表示正在尝试发送和接收消息。我们想循环遍历每个连接的客户端，将消息发送给它们。如果由于某些原因，通道被阻塞或消息无法发送，我们会认为这个客户端已断开连接，然后将其删除。

为了使代码简洁，创建一个manager.send方法循环遍历每个客户端。

```go
func (manager *ClientManager) send(message []byte, ignore *Client) {
    for conn := range manager.clients {
        if conn != ignore {
            conn.send <- message
        }
    }
}
```

至于conn.send如何发送数据，会在后面探讨。

现在我们可以探索 goroutine 如何读取客户端发送的websocket数据。这个goroutine的关键部分是读取socket数据，并将数据添加到manager.boradcast做进一步处理。

```go
func (c *Client) read() {
    defer func() {
        manager.unregister <- c
        c.socket.Close()
    }()
 
    for {
        _, message, err := c.socket.ReadMessage()
        if err != nil {
            manager.unregister <- c
            c.socket.Close()
            break
        }
        jsonMessage, _ := json.Marshal(&Message{Sender: c.id, Content: string(message)})
        manager.broadcast <- jsonMessage
    }
}
```

如果读取websocket数据时出错，可能意味着客户端已经断开连接。如果是这样，我们需要从服务器中注销这个客户端。

还记得前边的conn.send吗，它用来在第三个goroutine中写数据。

```go
func (c *Client) write() {
    defer func() {
        c.socket.Close()
    }()
 
    for {
        select {
        case message, ok := <-c.send:
            if !ok {
                c.socket.WriteMessage(websocket.CloseMessage, []byte{})
                return
            }
 
            c.socket.WriteMessage(websocket.TextMessage, message)
        }
    }
}
```

如果c.send通道具有数据，我们将尝试发送这些信息。如果由于某些原因，channel运行不正常，我们将向客户端发送断开连接的消息。

那么，如何启动这些goroutine呢，当我们启动服务器时，服务器goroutine将会启动，当有客户端连接时，其他goroutine将会启动。

例如，main方法中的代码：

```go
//$GOPATH/src/github.com/nraboy/realtime-chat/main.goGo
func main() {
    fmt.Println("Starting application...")
    go manager.start()
    http.HandleFunc("/ws", wsPage)
    http.ListenAndServe(":12345", nil)
}
```

我们在12345端口启动服务器，通过websocket连接访问。名为wsPage的方法如下所示：

```go
//$GOPATH/src/github.com/nraboy/realtime-chat/main.goGo
func wsPage(res http.ResponseWriter, req *http.Request) {
    conn, error := (&websocket.Upgrader{CheckOrigin: func(r *http.Request) bool { return true }}).Upgrade(res, req, nil)
    if error != nil {
        http.NotFound(res, req)
        return
    }
    client := &Client{id: uuid.NewV4().String(), socket: conn, send: make(chan []byte)}
 
    manager.register <- client
 
    go client.read()
    go client.write()
}
```

使用websocket包将HTTP请求升级到websocket请求。通过添加 CheckOrigin ，我们可以接受来自外部域的请求，从而消除跨域资源共享（CORS）的错误。

创建连接后，将创建一个客户端，并生成唯一的ID。如前所述，该客户端已经注册到服务器。客户端注册后，读写goroutine将被触发。

此时，我们可以通过如下命令启动应用。

```shell
go run *.go
```

你不能在web浏览器中测试，但是可以建立一个websocket连接到ws://localhost:12345/ws。

#### 创建Angular2 聊天客户端

 现在我们需要创建一个客户端的应用，我们可以发送和接收消息。假设您已经安装了[Angular 2 CLI](https://cli.angular.io/)，请执行以下操作：

```shell
ng new SocketExample
```

执行完将会生成一个单页应用，而我们想要完成的内容，是下方的动图演示的这样。

【图片】

JavaScript的websocket在Angular 2提供的一个类中。使用 Angular 2 CLI，通过执行如下操作创建提供程序（provider）。

```shell
ng g service socket	
```

上述命令会在您的项目中创建 src/app/socket.service.ts 和 src/app/socket.service.spec.ts 。spec文件用于单元测试，就不在本例中探索了。打开 src/app/socket.service.ts 文件，编写以下 TypeScript 代码：

```typescript
import { Injectable, EventEmitter } from '@angular/core';
 
@Injectable()
export class SocketService {
 
    private socket: WebSocket;
    private listener: EventEmitter<any> = new EventEmitter();
 
    public constructor() {
        this.socket = new WebSocket("ws://localhost:12345/ws");
        this.socket.onopen = event => {
            this.listener.emit({"type": "open", "data": event});
        }
        this.socket.onclose = event => {
            this.listener.emit({"type": "close", "data": event});
        }
        this.socket.onmessage = event => {
            this.listener.emit({"type": "message", "data": JSON.parse(event.data)});
        }
    }
 
    public send(data: string) {
        this.socket.send(data);
    }
 
    public close() {
        this.socket.close();
    }
 
    public getEventListener() {
        return this.listener;
    }
 
}
```

该提供者是可以注射的，并在触发某些事件事发送数据。在构造方法中，建立了与Go应用的WebSocket 连接，并创建了3个事件监听器。分别对应每个socket创建和销毁时，及接收到消息时。

send方法允许我们向Go应用发送消息，close方法用于通知Go应用我们将断开连接。

提供者程序已创建，但是还不能在我们的的应用程序的任何文件中使用。因此，我们需要将其添加到 src/app/app.module.ts 文件的 @NgModule 块中。打开文件并输入：

```typescript
import { BrowserModule } from '@angular/platform-browser';
import { NgModule } from '@angular/core';
import { FormsModule } from '@angular/forms';
import { HttpModule } from '@angular/http';
 
import { AppComponent } from './app.component';
import { SocketService } from "./socket.service";
 
@NgModule({
    declarations: [
        AppComponent
    ],
    imports: [
        BrowserModule,
        FormsModule,
        HttpModule
    ],
    providers: [SocketService],
    bootstrap: [AppComponent]
})
export class AppModule { }
```

需要注意的是，此时我们已经将provider导入并且添加到 @NgModule 块的 providers数组中了。

现在我们可以专注处理页面的逻辑了。打开 src/app/app.component.ts 文件，并输入以下代码：

```typescript
import { Component, OnInit, OnDestroy } from '@angular/core';
import { SocketService } from "./socket.service";
 
@Component({
    selector: 'app-root',
    templateUrl: './app.component.html',
    styleUrls: ['./app.component.css']
})
export class AppComponent implements OnInit, OnDestroy {
 
    public messages: Array<any>;
    public chatBox: string;
 
    public constructor(private socket: SocketService) {
        this.messages = [];
        this.chatBox = "";
    }
 
    public ngOnInit() {
        this.socket.getEventListener().subscribe(event => {
            if(event.type == "message") {
                let data = event.data.content;
                if(event.data.sender) {
                    data = event.data.sender + ": " + data;
                }
                this.messages.push(data);
            }
            if(event.type == "close") {
                this.messages.push("/The socket connection has been closed");
            }
            if(event.type == "open") {
                this.messages.push("/The socket connection has been established");
            }
        });
    }
 
    public ngOnDestroy() {
        this.socket.close();
    }
 
    public send() {
        if(this.chatBox) {
            this.socket.send(this.chatBox);
            this.chatBox = "";
        }
    }
 
    public isSystemMessage(message: string) {
        return message.startsWith("/") ? "<strong>" + message.substring(1) + "</strong>" : message;
    }
 
}
```

在上述 AppComponent类的构造方法中，我们注册我们的服务提供者并初始化需要绑定到UI的变量。在构造函数中加载或订阅是个不错的注意，我们使用ngOninit方法来完成。

```typescript
public ngOnInit() {
    this.socket.getEventListener().subscribe(event => {
        if(event.type == "message") {
            let data = event.data.content;
            if(event.data.sender) {
                data = event.data.sender + ": " + data;
            }
            this.messages.push(data);
        }
        if(event.type == "close") {
            this.messages.push("/The socket connection has been closed");
        }
        if(event.type == "open") {
            this.messages.push("/The socket connection has been established");
        }
    });
}
```

在上述方法中，我们订阅了在provider中创建的事件监听器。在这里我们需要检查发生了什么事件。如果是一条消息，需要检查是否存在发件人，然后将其添加到消息中。

你可能注意到了，一些消息是以斜线开始的。用来表示系统消息，稍后会将其加粗。

当客户端断开时，关闭事件将会发送到服务器，如果消息已经发送，它也会被发送到服务器。

在查看HTML之前，先添加一些CSS，使其看起来更像一个聊天应用。打开 src/style.css，输入以下内容：

```css
/* You can add global styles to this file, and also import other style files */
* { margin: 0; padding: 0; box-sizing: border-box; }
body { font: 13px Helvetica, Arial; }
form { background: #000; padding: 3px; position: fixed; bottom: 0; width: 100%; }
form input { border: 0; padding: 10px; width: 90%; margin-right: .5%; }
form button { width: 9%; background: rgb(130, 224, 255); border: none; padding: 10px; }
#messages { list-style-type: none; margin: 0; padding: 0; }
#messages li { padding: 5px 10px; }
#messages li:nth-child(odd) { background: #eee; }
```

现在，需要处理下HTML了。打开 src/app/app.component.html文件，并输入以下内容：

```html
<ul id="messages">
    <li *ngFor="let message of messages">
        <span [innerHTML]="isSystemMessage(message)"></span>
    </li>
</ul>
<form action="">
    <input [(ngModel)]="chatBox" [ngModelOptions]="{standalone: true}" autocomplete="off" />
    <button (click)="send()">Send</button>
</form>
```

这里我们只是简单的将消息数组遍历到屏幕上 。以斜线开头的消息将会被加粗。提交按钮绑定到了send方法中，当按下时，会提交输入框中的内容到Go应用。

#### 结语

刚刚演示了如何使用Go和Angular 2 创建一个WebSocket实时聊天应用。虽然没有在这个示例中存储聊天记录，但是这套逻辑可言应用于更复杂的项目，比如游戏，IOT，和其他很多场景。