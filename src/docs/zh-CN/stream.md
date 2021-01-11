# 流式API

通过流式API，您可以实时接收各种信息（例如，你的时间线中的新帖文，收到的消息，关注等），并进行各种操作。

## 连接到流

要使用流式API，您需要使用**websocket**连接到Misskey服务器。

请使用参数`i`连接到以下URL，并在websocket连接中包含认证信息。例如：
```
%WS_URL%/streaming?i=xxxxxxxxxxxxxxx
```

认证信息是您的API密钥，从应用程序连接到流时需要引用的用户访问令牌

<div class="ui info">
    <p><i class="fas fa-info-circle"></i> 关于如何获取认证信息，请参考<a href="./api">此文档</a>。</p>
</div>

---

您可以省略身份验证信息。此时无需登录即可使用，但是可以接收的信息和可以执行的操作将受到限制。例：

```
%WS_URL%/streaming
```

---

通过连接到流，您可以执行后文所示的API操作并订阅帖子。 但是此时例如时间线上的新帖子等还无法接收到。 要实现此功能，您需要连接到后文所述的流的**频道**。

**所有流交互都是JSON格式。**

## 频道
频道是Misskey的流API中的概念。这是一种分离发送和接收信息的机制。 您无法仅通过连接到Misskey流来实时接收时间线帖子。 需要通过连接到流中的频道，您才能够接收和发送各种消息。

### 连接到频道
要连接到频道，请将JSON数据发送到流：

```json
{
    type: 'connect',
    body: {
        channel: 'xxxxxxxx',
        id: 'foobar',
        params: {
            ...
        }
    }
}
```

其中：
* `channel`中可以设置您要连接的频道名。频道类型将在后面说明。
* `id`设置用于与频道通信的ID。因为流中有着各种消息，因此需要确定消息来自哪个频道。该ID可以是UUID或随机数。
* `params`是连接到频道时传的参数。连接不同的频道时需要不同的参数。连接到无需参数的频道时，该属性为可选。

<div class="ui info">
    <p><i class="fas fa-info-circle"></i> ID对应的是“频道的连接”，而不是频道。因为在某些情况下会使用不同的参数对同一频道进行多个连接。</p>
</div>

### 从频道接收消息
例如，当有新帖子时，时间线的频道将发送一条消息。通过接收此消息，您可以实时知道时间线上有新帖子。

当频道发出消息时，以下数据将以JSON格式传输到流中：
```json
{
    type: 'channel',
    body: {
        id: 'foobar',
        type: 'something',
        body: {
            some: 'thing'
        }
    }
}
```

其中：
* `id`为前文所述连接到频道时所设置的ID。因此可以知道此消息来自哪个频道。
* `type`为所设的消息类型。不同的频道会有不同类型的消息。
* `body`为所设的消息内容。不同的频道中的消息内容也会有不同。

### 向频道发送消息
根据频道的不同，您不仅可以接收消息，而且还可以发送消息并执行某些操作。

要将消息发送到频道，请将JSON格式数据发送到流：
```json
{
    type: 'channel',
    body: {
        id: 'foobar',
        type: 'something',
        body: {
            some: 'thing'
        }
    }
}
```

其中：
* `id`为前文所述连接到频道时想要设置的ID。因此您可以决定此消息发送到哪个频道。
* `type`为想要设置的消息类型。不同的频道会接受不同类型的消息。
* `body`为想要设置的消息内容。不同的频道接受的消息内容也会不同。

### 断开频道连接
要断开与频道的连接，请将JSON格式数据发送到流：

```json
{
    type: 'disconnect',
    body: {
        id: 'foobar'
    }
}
```

其中：
* `id`为前文所述连接到频道时想要设置的ID。

## ストリームを経由してAPIリクエストする

ストリームを経由してAPIリクエストすると、HTTPリクエストを発生させずにAPIを利用できます。そのため、コードを簡潔にできたり、パフォーマンスの向上を見込めるかもしれません。

ストリームを経由してAPIリクエストするには、次のようなデータをJSONでストリームに送信します:
```json
{
    type: 'api',
    body: {
        id: 'xxxxxxxxxxxxxxxx',
        endpoint: 'notes/create',
        data: {
            text: 'yee haw!'
        }
    }
}
```

其中：
* `id`には、APIのレスポンスを識別するための、APIリクエストごとの一意なIDを設定する必要があります。UUIDや、簡単な乱数のようなもので構いません。
* `endpoint`には、あなたがリクエストしたいAPIのエンドポイントを指定します。
* `data`には、エンドポイントのパラメータを含めます。

<div class="ui info">
    <p><i class="fas fa-info-circle"></i> APIのエンドポイントやパラメータについてはAPIリファレンスをご確認ください。</p>
</div>

### レスポンスの受信

APIへリクエストすると、レスポンスがストリームから次のような形式で流れてきます。

```json
{
    type: 'api:xxxxxxxxxxxxxxxx',
    body: {
        ...
    }
}
```

其中：
* `xxxxxxxxxxxxxxxx`の部分には、リクエストの際に設定された`id`が含まれています。これにより、どのリクエストに対するレスポンスなのか判別することができます。
* `body`には、レスポンスが含まれています。

## 投稿のキャプチャ

Misskeyは投稿のキャプチャと呼ばれる仕組みを提供しています。これは、指定した投稿のイベントをストリームで受け取る機能です。

例えばタイムラインを取得してユーザーに表示したとします。ここで誰かがそのタイムラインに含まれるどれかの投稿に対してリアクションしたとします。

しかし、クライアントからするとある投稿にリアクションが付いたことなどは知る由がないため、リアルタイムでリアクションをタイムライン上の投稿に反映して表示するといったことができません。

この問題を解決するために、Misskeyは投稿のキャプチャ機構を用意しています。投稿をキャプチャすると、その投稿に関するイベントを受け取ることができるため、リアルタイムでリアクションを反映させたりすることが可能になります。

### 投稿をキャプチャする

投稿をキャプチャするには、ストリームに次のようなメッセージを送信します:

```json
{
    type: 'subNote',
    body: {
        id: 'xxxxxxxxxxxxxxxx'
    }
}
```

其中：
* `id`にキャプチャしたい投稿の`id`を設定します。

このメッセージを送信すると、Misskeyにキャプチャを要請したことになり、以後、その投稿に関するイベントが流れてくるようになります。

例えば投稿にリアクションが付いたとすると、次のようなメッセージが流れてきます:

```json
{
    type: 'noteUpdated',
    body: {
        id: 'xxxxxxxxxxxxxxxx',
        type: 'reacted',
        body: {
            reaction: 'like',
            userId: 'yyyyyyyyyyyyyyyy'
        }
    }
}
```

其中：
* `body`内の`id`に、イベントを発生させた投稿のIDが設定されます。
* `body`内の`type`に、イベントの種類が設定されます。
* `body`内の`body`に、イベントの詳細が設定されます。

#### イベントの種類

##### `reacted`
その投稿にリアクションがされた時に発生します。

* `reaction`に、リアクションの種類が設定されます。
* `userId`に、リアクションを行ったユーザーのIDが設定されます。

例：
```json
{
    type: 'noteUpdated',
    body: {
        id: 'xxxxxxxxxxxxxxxx',
        type: 'reacted',
        body: {
            reaction: 'like',
            userId: 'yyyyyyyyyyyyyyyy'
        }
    }
}
```

##### `deleted`
その投稿が削除された時に発生します。

* `deletedAt`に、削除日時が設定されます。

例：
```json
{
    type: 'noteUpdated',
    body: {
        id: 'xxxxxxxxxxxxxxxx',
        type: 'deleted',
        body: {
            deletedAt: '2018-10-22T02:17:09.703Z'
        }
    }
}
```

##### `pollVoted`
その投稿に添付されたアンケートに投票された時に発生します。

* `choice`に、選択肢IDが設定されます。
* `userId`に、投票を行ったユーザーのIDが設定されます。

例：
```json
{
    type: 'noteUpdated',
    body: {
        id: 'xxxxxxxxxxxxxxxx',
        type: 'pollVoted',
        body: {
            choice: 2,
            userId: 'yyyyyyyyyyyyyyyy'
        }
    }
}
```

### 投稿のキャプチャを解除する

その投稿がもう画面に表示されなくなったりして、その投稿に関するイベントをもう受け取る必要がなくなったときは、キャプチャの解除を申請してください。

次のメッセージを送信します:

```json
{
    type: 'unsubNote',
    body: {
        id: 'xxxxxxxxxxxxxxxx'
    }
}
```

其中：
* `id`にキャプチャを解除したい投稿の`id`を設定します。

このメッセージを送信すると、以後、その投稿に関するイベントは流れてこないようになります。

# チャンネル一覧
## `main`
アカウントに関する基本的な情報が流れてきます。このチャンネルにパラメータはありません。

### 流れてくるイベント一覧

#### `转发`
自分の投稿がRenoteされた時に発生するイベントです。自分自身の投稿をRenoteしたときは発生しません。

#### `mention`
誰かからメンションされたときに発生するイベントです。

#### `readAllNotifications`
自分宛ての通知がすべて既読になったことを表すイベントです。このイベントを利用して、「通知があることを示すアイコン」のようなものをオフにしたりする等のケースが想定されます。

#### `meUpdated`
自分の情報が更新されたことを表すイベントです。

#### `follow`
自分が誰かをフォローしたときに発生するイベントです。

#### `unfollow`
自分が誰かのフォローを解除したときに発生するイベントです。

#### `followed`
自分が誰かにフォローされたときに発生するイベントです。

## `homeTimeline`
ホームタイムラインの投稿情報が流れてきます。このチャンネルにパラメータはありません。

### 流れてくるイベント一覧

#### `note`
タイムラインに新しい投稿が流れてきたときに発生するイベントです。

## `localTimeline`
ローカルタイムラインの投稿情報が流れてきます。このチャンネルにパラメータはありません。

### 流れてくるイベント一覧

#### `note`
ローカルタイムラインに新しい投稿が流れてきたときに発生するイベントです。

## `hybridTimeline`
ソーシャルタイムラインの投稿情報が流れてきます。このチャンネルにパラメータはありません。

### 流れてくるイベント一覧

#### `note`
ソーシャルタイムラインに新しい投稿が流れてきたときに発生するイベントです。

## `globalTimeline`
グローバルタイムラインの投稿情報が流れてきます。このチャンネルにパラメータはありません。

### 流れてくるイベント一覧

#### `note`
グローバルタイムラインに新しい投稿が流れてきたときに発生するイベントです。