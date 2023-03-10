# sox_mqtt

MQTTプロトコル上でSOXの時のようなセンサデータ配信を実現するためのライブラリです。
共通して使うことで、データフォーマットが定まり連携がしやすくなると嬉しいです。

git cloneして使ってください

```
pip install -r requirements.txt
```
sox_mqtt.pyをimportすることでmqttのpublishとsubscribeを支援します。

# MQTT Brokerの設定
- 最低限必要なこと
  - Retainを有効にします
    - Retainはパブリッシュされたメッセージをブローカーに保存しておくことで、パブリッシュ時にサブスクライブしていなかったクライアントでもサブスクライブした時に保存されている対象トピックの最新パブリッシュデータを受信できるようになります。
  - Retain Messageの保管場所をRAMではなく、discにします
    - RAMだとサーバが落ちた時にデータが消えてしまうのと、メモリサイズ分しかretainできません。
- 他のパラメータは用途に応じて調整してください

## Connection
- Connectionオブジェクトを用いて接続するブローカへの設定を行う
- connectメソッドによって設定した接続情報の入力確認を行い、接続に成功するとmqtt_clientを戻り値で返す
- client_idが複数クライアントで重複すると重複している者同士でコネクションを奪い合ってしまい、変な挙動を起こします(EMQXで確認)
    - 引数で指定しなかった場合、内部的に20桁の英数字の文字列をランダムで生成して設定するようにしている(__randomIdGenerateメソッド参照)のでほぼ重複はないはずです。
```
connection = Connection(broker_name, port, keepalive=60, client_id=randomId, username=None, password=None)
client = connection.connect()
```


## Create
- メタノードとしてそのトピックが扱う情報をretainerに登録します。
- メタノード名はpublishの宛先となるノード名_metaとして定義されます。

```python:create.py
publisher = PublishModule(client) # publishモジュールを通す
node = Node(node_name)
node.setLocation(lat, lng)
node.setDescription(description_contents)
```

``` python:create.py
transducer = node.Transducer(transducer_name)
transducer.setUnit(unit)
transducer.setMinValue(int(min_value))
transducer.setMaxValue(int(max_value))
transducer.setDescription(description_contens)
node.appendTransducer(transducer)
```

- ノードに情報を設定した後にcreateメソッドで作成する
``` python
publisher.create(node)
```


## Publish
- パブリッシャはNodeオブジェクトにデータをセットした上で、publishメソッドを用いてデータを配信します

``` python:publisher.py
publisher = PublishModule(client)
node = Node(node_name)
transducer = node.Transducer(transducer_name)
transducer.setRawValue(value) # 値を設定する
node.appendTransducer(transducer)
```

- publish時にはqosを`0`,`1`,`2`から設定可能
    - デフォルト値は`0`
    - QoSの値が大きくなるほど確実にサブスクライバにデータが送信されるような処理が加わります。
    - 優先度はPublisher QoS > Subscriber QoSです。(subscriberの方がQoSの設定値が低い場合、Publisher QoSに合わせてダウングレードするので注意)

``` python:publisher.py
publisher.publish(node, qos=0)
```

## Subscribe
- サブスクライバはsubscribeメソッドでnode_nameを指定してサブスクライブ登録をする
- SubscribeModuleを継承したオリジナルクラスを定義し、setProcessOnMessageメソッドでデータ受信時の処理を定義する。
- mqttにはワイルドカードとして`#`と`+`が用意されています。
- `#`を使った場合、指定した階層以下のすべてのトピックを同時にサブスクライブできます。
    - 例えば、`sensor/test1`, `sensor/test2`というトピックがあった時、`sensor/#`と指定することで、`sensor/`以下のすべてのトピックを同時にサブスクライブできます。
- `+`は、指定した階層の直下分だけ、なんでも良いとします。
    - 例えば、`sensor/test1/environment/`, `sensor/test2/environment`というのがあった時、`sensor/+/environment`とすることで、`sensor/自由なトピック名/environment`に一致するトピックからデータ取得します。
- トピックグルーピングに活用してください。 (sox_mqttjsも対応)
- *注意*: ワイルドカード演算子はサブスクライブにのみ使えます。

``` python:subscriber.py
# サブスクライブした時の処理をOverrideする方法
class OriginalSubscribeModule(SubscribeModule):
    def __init__(self, client):
        super().__init__(client)

    def setProcessOnMessage(self):
        def on_message(client, userdata, msg):
            # データ受信時の処理内容を実装する
        self.client.on_message = on_message
```

- subscribe時にはqosを`0`,`1`,`2`から設定可能
    - デフォルト値は`0`

``` python:subscriber.py
subscriber = OriginalSubscribeModule(client) # 独自クラスを作った場合
subscriber.subscribe(node_name, qos=0)   # subscribeするノードを指定する
subscriber.setProcessOnMessage()  # データ受信時の処理を設定
subscriber.run()                  # サブスクライブ開始
```

- データは以下のようにjson形式のテキストで送られてきます。json.loads()してtransducer名をkeyにアクセスしてデータを処理してください。
```
{
    transducer1_name : {
        raw_value: value,
        publish_timestamp: str(datetime)  
    },
   transducer2_name : {
        raw_value: value,
        publish_timestamp: str(datetime)  
    }, ... 
}
```


## Node
|要素名|説明|
|----|----|
|node_name|ノード名|
|location|位置情報(緯度,経度)の順で指定|
|transducers|ノードが扱うtransducer群|
|description|備考|
|create_timestamp|作成日時|

## Transducer
- メタノード


|要素名|説明|
|----|----|
|transducer_name|Transducer名|
|unit|単位|
|min_value|想定される最小値|
|max_value|想定される最大値|
|description|備考|

- データノード

|要素名|説明|
|----|----|
|node_name|ノード名(宛先指定に使う)|
|transducer|Transducer名: {rawValue: データ値, publish_timestamp: 配信日時} |