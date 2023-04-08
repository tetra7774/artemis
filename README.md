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

## DEMO

後で書く

## Broker側で起動までに行った内容  
### ユーザの作成  
artemisを起動後にまずはユーザを作成したかった。  
管理コンソールがあるのでそちらにログインする。  
初期のユーザ名とパスワードは「artemis」だった。
http://localhost:8161/console/auth/login
<img width="1279" alt="スクリーンショット 2023-03-26 20 49 50" src="https://user-images.githubusercontent.com/103823940/227773718-c7fe8816-8208-48e7-b2b1-dc34d5fa2108.png">

ArtemisタブのOperationにユーザを作成するAPIが提供されていた。ここで作成する。
<img width="1434" alt="スクリーンショット 2023-03-26 20 55 59" src="https://user-images.githubusercontent.com/103823940/227773977-084b2274-9d49-4e2a-8494-15a4d0b0d806.png">
  
ここで作成したユーザはConsoleでログインする時やClientからConnectionする時に利用した。  

### アドレスの作成 
pub/subする際にClientから指定する際に利用した。下記イメージのAPIで作成可能。
<img width="827" alt="スクリーンショット 2023-04-08 18 10 44" src="https://user-images.githubusercontent.com/103823940/230713564-51e59866-076d-491a-826b-738353820d4e.png">

### Clientの作成  
ClientはPythonを使って作成した。ライブラリはpahoを利用。  
- ライブラリインストール  
```
pip install paho-mqtt 
```
- Pythonのコード
```
import paho.mqtt.client as mqtt

# 接続情報を設定
broker_address = "artemis"  # Artemisのアドレス
port = 1883  # Artemisのポート番号
username = "user4"  # ユーザー名
password = "myPassword"  # パスワード

# MQTTクライアントを作成
client = mqtt.Client()
#client = mqtt.Client(protocol=mqtt.MQTTv5)
# 認証情報を設定
client.username_pw_set(username=username, password=password)

# MQTTブローカーに接続
client.connect(broker_address, port=port)

# Publishを実行
topic = "test"
payload = "Hello, world!!!"
client.publish(topic, payload)
print('this is a pen')
# Subscribeを実行
def on_message(client, userdata, message):
    print("Received message: ", str(message.payload.decode("utf-8")))

topic = "test"
client.subscribe(topic)
client.on_message = on_message

# メッセージを待機
client.loop_forever()
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
    




