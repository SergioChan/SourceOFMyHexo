title: 使用 Photon 在 Unity 里快速搭建一个多人联机游戏
date: 2016-08-18 16:17:02
categories: 闲言碎语


tags: [Unity, Photon]
---

如何最快的搭建一个 Unity 上的多人游戏？答案也许是自己搭建一个游戏服务器，也许是 LAN 解决方案，但是最快的解决方案还是使用成熟的第三方后端服务，我找到了 [Photon](https://www.photonengine.com/en-US/Photon)，一个看起来不错最后证实也挺靠谱的游戏后端解决方案。

一个游戏服务器，最主要的就是对不同参与者的事件同步和世界状态的同步，所以根本还是在于和服务器的长连接上，至于游戏中的用户体系，积分体系，货币体系这些，就是属于大后端的范畴了，也是可以独立于游戏同步服务器存在的系统。

### 安装

首先你需要集成 **Photon SDK For Unity**，下载地址在[这里](https://www.photonengine.com/en-US/Realtime/Download)。

![图1](/image/photon_1.png) 

你需要将下载下来的 `PhotoAssets` 中的所有文件都拖到你的项目的 `Assets` 文件夹里面，注意在 `Plugins` 里面有两个 `Photon3Unity3D.dll`，需要将 `Metro` 文件夹删掉，保留一个，否则在 Unity 编译的时候会报错。

拖进去之后只要 Unity 中 Compile 没有问题那第一步就大功告成了。

### 脚本创建

创建一个你用来维护游戏网络逻辑的脚本，例如命名为 `GameNetworkClient.cs`，然后你需要在头部加上这几个引用：

```
using System.Collections;
using ExitGames.Client.Photon;
using ExitGames.Client.Photon.LoadBalancing;
using System.Collections.Generic;
using UnityEngine;
using Hashtable = ExitGames.Client.Photon.Hashtable;
```

接着声明一个私有的 `LoadBalancingClient`对象，你的脚本中的所有网络逻辑都会通过这个 `client` 对象来发起和回调。我们这样在 Start 中初始化 `client` 对象：

```
client = new LoadBalancingClient();
client.AppId = "{你的 App_id}";  
client.OnStateChangeAction += this.OnStateChanged;
client.OnOpResponseAction += this.OnRespAction;
client.OnEventAction += this.OnEvent;

client.ConnectToRegionMaster("asia");
```

> 你的 App_id 可以在 Photon 官网注册完免费账户后在账户详情页获取，免费的账户拥有 **20** 个同时在线人数的限额，对于小范围好友间的游戏和测试足够了。

其中，最后一行代码就是 `client` 去连接 MasterServer 的方法，这里面传入的参数要根据你的游戏所在地区来确定， Photon 在全球有很多分散的数据中心，因此支持很多区域的连接，具体支持的地区和代码如下：

| Region        | Hosted in  | Token |
| ------------- | ---------- | ----- |
| Asia          | Singapore  | asia  |
| Australia     | Melbourne  | au    |
| Canada, East  | Montreal   | cae   |
| Europe        | Amsterdam  | eu    |
| Japan         | Tokyo      | jp    |
| South America | Sao Paulo  | sa    |
| USA, East     | Washington | us    |
| USA, West     | San José   | usw   |

所以在这里我改成了 `asia`，事实证明新加坡的服务器是比较稳定的。

### 状态维护

`client` 的状态主要就是根据上面初始化的时候给的三个 Action 来维护的，你需要在你的脚本里为这三个 Action 都加上你自己的 Handler。当然，如果你想按照他的 demo 里示范的那样去继承 `client` 并且 override 这三个 Action 的回调都是可以的。

#### OnStateChanged

返回的是一个 `ClientState` 枚举值，主要就是一些 client 状态的值，例如

 `connecting`，`connected`，`joining`之类的。

#### OnRespAction

返回的是一个 `OperationResponse` 对象，它会在每次你用 `client` 对象调用一些方法并且获得 response 之后调用，分别有下面这些类型的 Operation:

```
public class OperationCode {
        [Obsolete("Exchanging encrpytion keys is done internally in the lib now. Don't expect this operation-result.")]
        public const byte ExchangeKeysForEncryption = 250;

        /// <summary>(255) Code for OpJoin, to get into a room.</summary>
        public const byte Join = 255;

        /// <summary>(230) Authenticates this peer and connects to a virtual application</summary>
        public const byte Authenticate = 230;

        /// <summary>(229) Joins lobby (on master)</summary>
        public const byte JoinLobby = 229;

        /// <summary>(228) Leaves lobby (on master)</summary>
        public const byte LeaveLobby = 228;

        /// <summary>(227) Creates a game (or fails if name exists)</summary>
        public const byte CreateGame = 227;

        /// <summary>(226) Join game (by name)</summary>
        public const byte JoinGame = 226;

        /// <summary>(225) Joins random game (on master)</summary>
        public const byte JoinRandomGame = 225;

        // public const byte CancelJoinRandom = 224; // obsolete, cause JoinRandom no longer is a "process". now provides result immediately

        /// <summary>(254) Code for OpLeave, to get out of a room.</summary>
        public const byte Leave = (byte)254;

        /// <summary>(253) Raise event (in a room, for other actors/players)</summary>
        public const byte RaiseEvent = (byte)253;

        /// <summary>(252) Set Properties (of room or actor/player)</summary>
        public const byte SetProperties = (byte)252;

        /// <summary>(251) Get Properties</summary>
        public const byte GetProperties = (byte)251;

        /// <summary>(248) Operation code to change interest groups in Rooms (Lite application and extending ones).</summary>
        public const byte ChangeGroups = (byte)248;

        /// <summary>(222) Request the rooms and online status for a list of friends (by name, which should be unique).</summary>
        public const byte FindFriends = 222;

        /// <summary>(221) Request statistics about a specific list of lobbies (their user and game count).</summary>
        public const byte GetLobbyStats = 221;

        /// <summary>(220) Get list of regional servers from a NameServer.</summary>
        public const byte GetRegions = 220;

        /// <summary>(219) WebRpc Operation.</summary>
        public const byte WebRpc = 219;
}
```

可以看到，例如 `rasieEvent`，`setProperties` 或者加入离开游戏这些请求都是会有服务器的 response 的，你可以根据返回对象的 `ReturnCode` 来判断请求是否成功并且是否执行一些错误后的处理，`ReturnCode` 为 0 成功，不为 0 则失败，如果需要也可以从 `DebugMessage` 中获取错误提示信息。

#### **OnEventAction** 

返回的是一个 `EventData` 对象，它会在每次 `client` 接收到新的 Event 的时候调用，具体的 Event 类型根据`EventData` 的 `EventCode` 来确定，分别有下面这些类型的 `EventCode`:

```
public class EventCode {
        /// <summary>(230) Initial list of RoomInfos (in lobby on Master)</summary>
        public const byte GameList = 230;

        /// <summary>(229) Update of RoomInfos to be merged into "initial" list (in lobby on Master)</summary>
        public const byte GameListUpdate = 229;

        /// <summary>(228) Currently not used. State of queueing in case of server-full</summary>
        public const byte QueueState = 228;

        /// <summary>(227) Currently not used. Event for matchmaking</summary>
        public const byte Match = 227;

        /// <summary>(226) Event with stats about this application (players, rooms, etc)</summary>
        public const byte AppStats = 226;

        /// <summary>(224) This event provides a list of lobbies with their player and game counts.</summary>
        public const byte LobbyStats = 224;

        /// <summary>(210) Internally used in case of hosting by Azure</summary>
        [Obsolete("TCP routing was removed after becoming obsolete.")]
        public const byte AzureNodeInfo = 210;

        /// <summary>(255) Event Join: someone joined the game. The new actorNumber is provided as well as the properties of that actor (if set in OpJoin).</summary>
        public const byte Join = (byte)255;

        /// <summary>(254) Event Leave: The player who left the game can be identified by the actorNumber.</summary>
        public const byte Leave = (byte)254;

        /// <summary>(253) When you call OpSetProperties with the broadcast option "on", this event is fired. It contains the properties being set.</summary>
        public const byte PropertiesChanged = (byte)253;

        /// <summary>(253) When you call OpSetProperties with the broadcast option "on", this event is fired. It contains the properties being set.</summary>
        [Obsolete("Use PropertiesChanged now.")]
        public const byte SetProperties = (byte)253;

        /// <summary>(252) When player left game unexpected and the room has a playerTtl > 0, this event is fired to let everyone know about the timeout.</summary>
        /// Obsolete. Replaced by Leave. public const byte Disconnect = LiteEventCode.Disconnect;

        /// <summary>(251) Sent by Photon Cloud when a plugin-call or webhook-call failed. Usually, the execution on the server continues, despite the issue. Contains: ParameterCode.Info.</summary>
        /// <seealso cref="https://doc.photonengine.com/en/realtime/current/reference/webhooks"/>
        public const byte ErrorInfo = 251;

        /// <summary>(250) Sent by Photon whent he event cache slice was changed. Done by OpRaiseEvent.</summary>
        public const byte CacheSliceChanged = 250;
}
```

这里的 `EventCode` 你是可以自己定义的，这个 code 是个 **byte 类型的整数**，因此不能大于255，Photon 将从 0 开始的一大段值域都留给开发者用来自定义事件了。Photon 默认提供了 `GameList`，或者说 RoomList 的功能，你可以创建 `Room` 并且加入，`Room` 也有他自己的 Option，可以作为 Lobby，也就是所有人默认进入的 `Room`，当然也可以根据各种条件来查找 `Room`。

这里的 **253 PropertiesChanged** 是一个非常重要的 Event，在上面我提到过，联机游戏的后端最重要的部分之一就是世界状态的同步，在这里也就是**房间的状态 Room Properties**，因此在 Photon SDK 中，当你调用 `client` 对当前加入的 `Room` 的某个属性做出了改变，这就会产生一个事件通知到整个 `Room` 里的所有玩家，这通常用来同步一些全局的属性，例如光线，地形，怪物的血量之类的。

当然，你还可以自定义 Event，在我实现的 demo 中我就是用到了自定义 Event 来告知其他玩家我的状态。比如，整个地图 （可以看做就是一个 `Room`）中的玩家列表和位置是可以用全局状态来同步的，但是例如单个玩家的一些事件（使用物品，使用技能之类的）可能就需要由发起的用户向全 `Room` 的其他玩家发送一个同步事件。

```
byte eventCode = 1; 
Hashtable evData = new Hashtable (); 
evData.Add ("player_id", random_playerid);
evData.Add ("pos_x", position.x);
evData.Add ("pos_y", position.y);

bool sendReliable = true; 
if (isConnected) {
	client.OpRaiseEvent (eventCode, evData, sendReliable, RaiseEventOptions.Default);
}
```

> 然后这个事件就会在除了发送者自己的其他玩家的客户端被回调。

**这个时候收到的数据包怎么解出来呢？**

最后再说说坑了我半天的 `EventData` 中的数据抽取，在 `EventData` 中的数据格式十分蛋疼，例如上面这段代码里发送的数据，当你在其他客户端收到的时候，它的数据其实是存在 `EventData` 的 `Parameters` 这个属性变量里面，这是一个 `Dictionary<byte,object>` 类型的对象，通过 `ParameterCode.Data` 这个 key 将 data 取出来之后，里面才是我们传入的 Hashtable ，然而现在已经变成了一个 `Dictionary<object,object>` 对象了。在没有仔细查看源码的情况下，它的官方 demo 和文档并没有提及如何解包 `EventData` ,因此我是通过 Debug 的方式才最终摸清了里面的数据格式。当然后来又看了下它的头文件，才看到 `ParameterCode.Data` 😂。

