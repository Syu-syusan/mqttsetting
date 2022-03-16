# 目次
- [始めに](#始めに)
- [環境構築](#環境構築)
- [Thingsファイル](#thingsファイル)
- [Itemsファイル](#Itemsファイル)
- [Rulesファイル](#rulesファイル)

# 始めに
OpenHABの設定ファイルの記述方法を本ドキュメントで示す。Linuxにおける標準では、`/etc/openhab`　ディレクトリ配下にサブディレクトリが用意されているので、そこに各種ファイルを記述していく。  

例として、SESAMEスマートロックから取得できるデータをOpenHABでどう管理し、操作していくかを定義するファイルの記述方法を示す。  
GLC用に用意したシステム構成図を以下に示す。以後の説明は以下の図をもとにしている。  

![haudi-ページ1のコピー](https://user-images.githubusercontent.com/74150680/158243840-25faf139-c9f7-43d3-b002-a3fca38e8427.jpg)

# 環境構築
OpenHABのインストール方法は、本ドキュメントと同じプロジェクト配下にある`reTerminal_setup.md` を別途参照。  

インストール後、OpenHABのGUIから各種必要となるバインディング（アドオン）の追加をする必要がある。バインディングの追加に関しては、GUIからの追加を公式も推奨している。  
例として、スマートロックではなくTapo電球を操作する場合、Tapoアドオンの追加が必要となる。  
数多くのバインディングがあるが、後述にあるように、MQTTブローカを介してやり取りをしている場合には**MQTTバインディング**、JSON形式のデータを管理する際には、**JSONPath Transformationアドオン**が必須となる。  
また、VSCodeで作業を行う場合、拡張機能で用意されているOpenHABを入れることを推奨する。

# Thingsファイル
Thingsファイルでは、「**何**で接続された**どのモノ・トピック**」をOpenHABに接続するかを定義する。OpenHABに機器を追加するときには、常にThingsファイルの記述から始める。モノはバインディングを通じてOpenHABに接続されるため、追加するモノをつなげるバインディングの選択と、トピックの指定がが必要となる。  

スマートロックは、ローカルで動作しているPythonスクリプトを介してローカルのMQTTブローカにサブスクライブされているため、「**MQTTブローカ**で接続された**スマートロックとそのトピック**」を追加することになる。 

これをファイルで記述すると、以下のようになる。  
```
Bridge mqtt:broker:broker [ host="127.0.0.1", secure=false, username="haudidevices", password="vZSEiULxZub3oWGwEkMyFLr3", qos=0] {
    //DoorLock_Lock
    Thing topic DoorLock_Lock_Status_ui "DoorLock_Lock_Status_ui" {
    Channels:
      Type string : DoorLock_Lock_Status_ui [stateTopic="haudi/hc1.0/ul3fqMC4uNtvOOjGP5WCcw/ui/Home1_Room101_Lock1_Lock/status" , transformationPattern="JSONPATH:$.button"]
    }
    Thing topic DoorLock_Lock_Command_ui "DoorLock_Lock_Command_ui" {
    Channels:
      Type string : DoorLock_Lock_Command_ui [commandTopic="haudi/hc1.0/ul3fqMC4uNtvOOjGP5WCcw/ui/Home1_Room101_Lock1_Lock/command"]
    }
    Thing topic DoorLock_Lock_Status_device "DoorLock_Lock_Status_device" {
    Channels:
      Type string : DoorLock_Lock_Status_device [stateTopic="haudi/hc1.0/ul3fqMC4uNtvOOjGP5WCcw/lock/h1_r1_lock1/status", transformationPattern="JSONPATH:$.locked"]
    }
    Thing topic DoorLock_Lock_Command_device "DoorLock_Lock_Command_device" {
    Channels:
      Type string : DoorLock_Lock_Command_device [commandTopic="haudi/hc1.0/ul3fqMC4uNtvOOjGP5WCcw/lock/h1_r1_lock1/command"]
    }
    Thing topic DoorLock_Lock_Command_Request_ui "DoorLock_Lock_Command_Request_ui" {
    Channels:
      Type string : DoorLock_Lock_Command_Request_ui [stateTopic="haudi/hc1.0/ul3fqMC4uNtvOOjGP5WCcw/ui/Home1_Room101_Lock1_Lock/command_request"]
    }
    Thing topic DoorLock_Lock_Status_Request_device "DoorLock_Lock_Status_Request_device" {
    Channels:
      Type string : DoorLock_Lock_Status_Request_device [commandTopic="haudi/hc1.0/ul3fqMC4uNtvOOjGP5WCcw/lock/h1_r1_lock1/status_request"]
    }
}
```

1行目で、MQTTによってこの機器が接続されていることを定義している。何で接続されているかを定義する際には、`Bridge`を最初につける。  
`Bridge`の次の3つの記述(`mqtt:broker:broker`)は、MQTTで接続する際には必須の記述となる。  
次の角括弧内で、使用しているMQTTブローカに関する情報を記述する。上の例では、ローカルのMQTTブローカにユーザ名とパスワードを指定して接続をしている。  

ここまでが、MQTTに関する記述となっている。  

次に、扱うデータを定義する。データを定義する際には`Thing`をつける。  
`Thing`の次の3つの記述で、**どのデバイス**の**どのトピック**と分かるようにラベルを付けている。例の、一番初めでは`topic DoorLock_Lock_Status_ui "DoorLock_Lock_Status_ui"`として、UIのステータスを定義している。  
次の波括弧内で、扱うトピックの詳細を定義している。扱うトピックの詳細は**チャンネル**という単位で管理を行う。扱うトピックの詳細を定義する際には、`Channels:`を最初につける。チャンネルは、後述の**Items**と**Thing**をつなげる役割を果たす。  
次の記述で、扱うデータがどういったデータなのかを管理する。どのトピックなのか分かるようにラベルを付け、そのトピックを実際に指定している。# **haudiプロトコルに準拠、statetopicとcommandtopicを分ける**  
`transformationPattern="JSONPATH:$.button""`という記述をつけることによって、JSONデータから、特定の値のみを引き抜くことが可能となる。

# Itemsファイル
Itemsファイルでは、Thingsファイルで用意した、各Channelの**受け皿**の役割となるItemを定義する。Itemには状態があり、後述のruleやWebAppを通じて使用される。  

ファイルでは、データの型と、前述のThingsファイルで名付けたラベルなどを記述する。実際のファイルでは以下のような記述になっている。  

```
String DoorLock_Lock_Command_ui {channel="mqtt:topic:broker:DoorLock_Lock_Command_ui:DoorLock_Lock_Command_ui"}

String DoorLock_Lock_Status_ui {channel="mqtt:topic:broker:DoorLock_Lock_Status_ui:DoorLock_Lock_Status_ui"}

String DoorLock_Lock_Command_device {channel="mqtt:topic:broker:DoorLock_Lock_Command_device:DoorLock_Lock_Command_device"}

String DoorLock_Lock_Status_device {channel="mqtt:topic:broker:DoorLock_Lock_Status_device:DoorLock_Lock_Status_device"}

String DoorLock_Lock_Command_Request_ui {channel="mqtt:topic:broker:DoorLock_Lock_Command_Request_ui:DoorLock_Lock_Command_Request_ui"}

String DoorLock_Lock_Status_Request_device {channel="mqtt:topic:broker:DoorLock_Lock_Status_Request_device:DoorLock_Lock_Status_Request_device"}

String DoorLock_Battery_Status_device {channel="mqtt:topic:broker:DoorLock_Battery_Status_device:DoorLock_Battery_Status_device"}

String DoorLock_Battery_Command_ui {channel="mqtt:topic:broker:DoorLock_Battery_Command_ui:DoorLock_Battery_Command_ui"}

String DoorLock_Battery_Command_Request_ui {channel="mqtt:topic:broker:DoorLock_Battery_Command_Request_ui:DoorLock_Battery_Command_Request_ui"}

String AutoLock_Lock_Status_ui {channel="mqtt:topic:broker:AutoLock_Lock_Status_ui:AutoLock_Lock_Status_ui"}

String AutoLock_Lock_Command_ui {channel="mqtt:topic:broker:AutoLock_Lock_Command_ui:AutoLock_Lock_Command_ui"}

String AutoLock_Lock_Command_Request_ui {channel="mqtt:topic:broker:AutoLock_Lock_Command_Request_ui:AutoLock_Lock_Command_Request_ui"}
```

# Rulesファイル
Rulesファイルでは、ruleとしてある条件で一連のコマンドを実行することを定義できる。  
Javaの発展系のXtendで記述することになっている。  
例として、Webアプリで鍵の施錠の操作をした際に、実際に鍵の解錠を行うruleの記述を以下に示す。  

```
rule "DoorLock_LockFalse_OHtoMQTT2SESAME"
when
    Item DoorLock_Lock_Status_ui received update "false"
then
        DoorLock_Lock_Command_device.sendCommand('{"locked":false}')
end
```

ruleの記述には、**rule-when-then-end**の4つの要素が必要となる。  
1行目のruleの後の記述によって、ruleの名前を定義している。  
2行目のwhenの後の記述によって、そのruleがどの条件でコマンドを実行するようにするかを定義している。例では、Webアプリで鍵の開錠の操作（正確には、`haudi/hc1.0/ul3fqMC4uNtvOOjGP5WCcw/ui/Home1_Room101_Lock1_Lock/status`がfalseになった時）をした際に、コマンドを実行するようにしている。  
4行目のthenの後の記述によって、実行するコマンドの詳細を定義している。例では、デバイスの操作を行うトピック（`haudi/hc1.0/ul3fqMC4uNtvOOjGP5WCcw/lock/h1_r1_lock1/command`）にメッセージ（`'{"locked":false}'`）を送ることで、鍵の施錠を行うコマンドを定義している。  
応用として、rulesでは別のデバイスのItemを条件とすることも可能である。また、Xtendで記述することのできる任意のコマンド（メッセージ）を実行することが可能である。  
実際、オートロック機能はドアセンサーを条件に含めることで実装している。  
