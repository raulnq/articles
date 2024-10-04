---
title: "Using Raw WebSockets in .NET"
datePublished: Fri Oct 04 2024 20:58:23 GMT+0000 (Coordinated Universal Time)
cuid: cm1v7jc5h001809l3btkld5l7
slug: using-raw-websockets-in-net
cover: https://cdn.hashnode.com/res/hashnode/image/upload/v1728053488925/c9998d1f-a8c6-44fe-9ab5-47155256af82.png
tags: websockets, dotnet

---

Some time ago, in [Getting Started with SignalR in .NET 6](https://blog.raulnq.com/getting-started-with-signalr-in-net-6), we discussed how easy it is to add real-time features to our applications. However, in some situations, we might need something simple, like using the basic functionality offered by .NET to implement [WebSockets](https://learn.microsoft.com/en-us/aspnet/core/fundamentals/websockets?view=aspnetcore-8.0&preserve-view=true).

This time, we will implement a basic chat application. Run the following commands to set up the project and solution:

```powershell
dotnet new webapi -n ChatServer
dotnet new sln -n Websockets
dotnet sln add ChatServer
```

Open the solution and replace the content `Program.cs` file with the following content:

```csharp
using System.Net.WebSockets;
using System.Text;
using System.Collections.Concurrent;

var builder = WebApplication.CreateBuilder(args);
var app = builder.Build();
app.UseWebSockets();
var clients = new ConcurrentDictionary<Guid, WebSocket>();

app.Map("/chat", async context =>
{
    if (context.WebSockets.IsWebSocketRequest)
    {
        var webSocket = await context.WebSockets.AcceptWebSocketAsync();
        await Handle(webSocket, clients);
    }
    else
    {
        context.Response.StatusCode = StatusCodes.Status400BadRequest;
    }
});

app.Run();
```

To enable Websockets, we add the corresponding middleware using the `app.UseWebSockets()` statement. Then, we define an endpoint at the path `/chat` that detects web socket requests. The `HandleRequest` method is as follows:

```csharp
async Task HandleRequest(WebSocket webSocket, ConcurrentDictionary<Guid, WebSocket> clients)
{
    var clientId = Guid.NewGuid();
    clients.TryAdd(clientId, webSocket);

    while (webSocket.State == WebSocketState.Open)
    {
        var (payload, messageType) = await Read(webSocket);
        if (messageType == WebSocketMessageType.Close)
        {
            await webSocket.CloseAsync(webSocket.CloseStatus!.Value, webSocket.CloseStatusDescription, CancellationToken.None);
            break;
        }
        if (messageType == WebSocketMessageType.Text)
        {
            await Broadcast(payload, clientId, clients);
        }
    }

    webSocket.Dispose();
    clients.TryRemove(clientId, out _);
}
```

The connection between the server and the client will be managed through an infinite loop while the socket remains open. The basic algorithm will be as follows:

* Check if the web socket is still open.
    
* Read the data.
    
* If the message type is `Close`, we close the web socket.
    
* If the message type is `Text`, we broadcast the message to all the clients.
    

The `Read` method is as follows:

```csharp
async Task<(List<byte>, WebSocketMessageType)> Read(WebSocket webSocket)
{
    var buffer = new byte[1024 * 4];
    var payload = new List<byte>(1024 * 4);
    WebSocketReceiveResult? result;
    do
    {
        result = await webSocket.ReceiveAsync(new ArraySegment<byte>(buffer), CancellationToken.None);
        payload.AddRange(new ArraySegment<byte>(buffer, 0, result.Count));
    }
    while (result.EndOfMessage == false);
    return (payload, result.MessageType);
}
```

Since the client can send the message in several chunks, we keep iterating over `ReceiveAsync` until we reach the end of the message. The `Broadcast` method is as follows:

```csharp
async Task Broadcast(List<byte> payload, Guid clientId, ConcurrentDictionary<Guid, WebSocket> clients)
{
    var receivedMessage = Encoding.UTF8.GetString(payload.ToArray());
    foreach (var key in clients.Keys)
    {
        var message = Encoding.UTF8.GetBytes($"{clientId} says {receivedMessage}");
        await clients[key].SendAsync(new ArraySegment<byte>(message), WebSocketMessageType.Text, true, CancellationToken.None);
    }
}
```

We decode the payload in `UTF8`, create a new message, and then encode the result into a new message. Every message is sent to all the connected clients. To test our chat server, create a `client.html` file at the solution level with the following content:

```xml
<!DOCTYPE html>
<html>
<head>
    <meta charset="utf-8">
    <title>Chat</title>
</head>
<body>
    <input id="message" placeholder="Message" />
    <button id="send" type="button">Send</button>
    <hr />
    <ul id="messages"></ul>
    <script>
        document.addEventListener("DOMContentLoaded", () => {
            const websocket = new WebSocket("http://localhost:5263/chat");
            websocket.onmessage = (e) => {
                const li = document.createElement("li");
                li.textContent = `${e.data}`;
                document.getElementById("messages").appendChild(li);
            };
            document.getElementById("send").addEventListener("click", async () => {
                const message = document.getElementById("message").value;
                try {
                    await websocket.send(message);
                } catch (err) {
                    console.error(err);
                }
            });
        });
    </script>
</body>
</html>
```

The script above starts the WebSocket and sets up a handler to receive messages from the server. It also sets up a handler for the button to send messages to the server. You can find all the code [here](https://github.com/raulnq/websockets). Thanks, and happy coding.