# ActiveMQ（artemis）を使ってPub／Subしてみた

## 詳細
技術調査でActiveMQ(artemis)でMQTTv3.1.1とv5が併用して利用できるか調査することになった。  
やりたいこととしては、Brokerを立てて、Client側で指定したMQTTv3.1.1とv5でBrokerにConnectできるか。  
(もし、Broker側で何らか手を入れないとv3.1.1とv5の切り替えができないとかだと悲しい)

# Requirement
今回使用したライブラリを列挙する  
**Broker**  
* ActiveMQ Artemis 2.28.0
* Docker version 20.10.22

**Client**
* Python 3.6.15
* paho-mqtt 1.6.1
* Docker version 20.10.22

## Pub／SubのためにBrokerで行った設定 
### ユーザの作成  
管理コンソールでユーザやアドレスを作成できるのでまずはそちらにログイン。
    
初期のユーザ名とパスワードは「artemis」だった。
http://localhost:8161/console/auth/login
<img width="1279" alt="スクリーンショット 2023-03-26 20 49 50" src="https://user-images.githubusercontent.com/103823940/227773718-c7fe8816-8208-48e7-b2b1-dc34d5fa2108.png">

ArtemisタブのOperationにユーザを作成するAPIが提供されていた。ここで作成する。
<img width="1434" alt="スクリーンショット 2023-03-26 20 55 59" src="https://user-images.githubusercontent.com/103823940/227773977-084b2274-9d49-4e2a-8494-15a4d0b0d806.png">
   

### アドレスの作成 
pub/subする際にClientから指定する際に利用した。下記イメージのAPIで作成可能。
アドレスはメッセージの宛先となる。アドレスにキューがバインドされていて、そのキューの名前がTopicということになるのかな。。(アドレス宛に送信したメッセージがTopic名をプレフィックスとしたキューに投げ込まれてくことをイメージ)
※[参考](https://activemq.apache.org/components/artemis/documentation/)(MQTT保持メッセージ)

<img width="827" alt="スクリーンショット 2023-04-08 18 10 44" src="https://user-images.githubusercontent.com/103823940/230713564-51e59866-076d-491a-826b-738353820d4e.png">

### Clientの作成  
ClientはPythonを使って作成した。ライブラリはpahoを利用。  
- ライブラリインストール  
```
pip install paho-mqtt 
```
- Pythonのコード(Pub/Sub)
```
import paho.mqtt.client as mqtt
import time

# 接続情報
broker_address = "artemis"  # Artemisのホスト名
broker_port = 1883  # Artemisのポート番号
username = "user4"  # Artemisのユーザ名
password = "myPassword"  # Artemisのパスワード
client_id = "12345"  # クライアントID
address = "test"  # アドレス名（ActiveMQ Artemisで事前に作成されている必要があります）

# コールバック関数: メッセージ受信時に実行される
def on_message(client, userdata, message):
    print("受信したメッセージ:", message.payload.decode())

# コールバック関数: Publishの応答処理
def on_publish(client, userdata, mid):
    print("メッセージが正常にパブリッシュされました。")

# MQTTクライアントの初期化
client = mqtt.Client(client_id=client_id)

# 認証情報の設定
client.username_pw_set(username, password)

# コールバック関数をクライアントに割り当て
client.on_message = on_message
client.on_publish = on_publish

# Artemisに接続
client.connect(broker_address, broker_port)

# メッセージ受信のためのループ開始
client.loop_start()

# アドレスに対するトピックを作成
topic = address + "/test"

# サブスクライブ（トピックの購読）開始
client.subscribe(topic)

# メッセージのパブリッシュ
message = "Hello, Artemis!"
result, mid = client.publish(topic, payload=message)

# Publishの結果を確認
if result == mqtt.MQTT_ERR_SUCCESS:
    print("メッセージがパブリッシュされました。")
else:
    print("メッセージのパブリッシュに失敗しました。")

# 一定時間待機してメッセージを受信するための時間を与える
time.sleep(5)

# 接続の終了
client.loop_stop()
client.disconnect()

```

- Pythonのコード(Pub)
```
import paho.mqtt.client as mqtt
import time

# 接続情報
broker_address = "artemis"  # Artemisのホスト名
broker_port = 1883  # Artemisのポート番号
username = "user4"  # Artemisのユーザ名
password = "myPassword"  # Artemisのパスワード
client_id = "12345"  # クライアントID
topic = "test"  # パブリッシュするトピック

# コールバック関数: Publishの応答処理
def on_publish(client, userdata, mid):
    print("メッセージが正常にパブリッシュされました。")

# MQTTクライアントの初期化
client = mqtt.Client(client_id=client_id)

# 認証情報の設定
client.username_pw_set(username, password)

# コールバック関数をクライアントに割り当て
client.on_publish = on_publish

# Artemisに接続
client.connect(broker_address, broker_port)

# メッセージのパブリッシュ
message = "Hello, Artemis!"

# 10回のループでメッセージをパブリッシュ
for i in range(10):
    # メッセージをパブリッシュ
    result, mid = client.publish(topic, payload=message)

    # Publishの結果を確認
    if result == mqtt.MQTT_ERR_SUCCESS:
        print("メッセージがパブリッシュされました。")
    else:
        print("メッセージのパブリッシュに失敗しました。")

    time.sleep(1)  # 1秒間の待機

# 接続の終了
client.disconnect()

```

### Broker内の設定（特に見てた設定ファイル）
Artemis内の設定ファイルのうち、見てたファイルとざっくりメモを書いとく。間違っているかもなのでそこは注意、  
- broker.xml
    - artemisを構成するファイルのうち、一番主要な設定が乗っているファイルらしい。「security-settings」を見ると、どのroleにどの権限を割り当てるかの設定。  
    「artemis-roles.properties」でユーザとroleの紐付けが行われている。
    - ActiveMQではアクセプターという概念があり、このアクセプター(MQTT)に対して、MQTTのVersionを指定することができるよう。  
```
<!-- MQTT Acceptor -->
     <acceptor name="mqtt">tcp://0.0.0.0:1883?tcpSendBufferSize=1048576;tcpReceiveBufferSize=1048576;protocols=MQTT;useEpoll=true</acceptor>
    #<acceptor name="mqtt">tcp://0.0.0.0:1883?protocols=MQTT,MQTTV3.1,MQTTV3.1.1,MQTTV5;useEpoll=true</acceptor>   
``` 
- artemis-users.properties  
作成したユーザはこの設定ファイルで管理されている。  
「ユーザ名＝PW」の形で管理されてる。PWは平文かハッシュ化するか選択可能。
- artemis.profile  
ここで見てたのは下記の箇所。hawtioはjavaアプリケーションにコンソールを作成するためのモジュールらしく、ActiveMQもhawtioが使われているっぽい。ここでは、コンソールにログインできるroleを指定している。  
```
HAWTIO_ROLE='amq,amq2'
```

# 参考文献
[MQTT Version 3.1.1 をふりかえる](https://tech-blog.optim.co.jp/entry/2019/08/20/163000)  
[MQTT Version 5.0 でできること](https://tech-blog.optim.co.jp/entry/2020/09/25/110000)   
[Apache ActiveMQ Artemis 2.28.0 User Manual](https://activemq.apache.org/components/artemis/documentation/)  
- ユーザマニュアルで参考になった章
    - [MQTT](https://activemq.apache.org/components/artemis/documentation/2.16.0/mqtt.html)⇨この章にartemisはMQTTv3.1.1とv5の両方に対応していることが記載    

[Apache ActiveMQ Artemis 2.28.0](https://github.com/apache/activemq-artemis/tree/main/artemis-docker)   
[paho](https://github.com/eclipse/paho.mqtt.python) 
    




