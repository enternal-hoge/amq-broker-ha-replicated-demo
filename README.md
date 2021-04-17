# amq-broker-ha-replicated-demo

## 前提条件

ローカルPCでAMQ Brker HA Replicatedを試す。

## 実行

```
$ ./init.sh 

#################################################################
##                                                             ##
##  Setting up the JBoss AMQ 7 Replicated HA Demo              ##
##                                                             ##
##                                                             ##
##                   ###    ##     ##  #######                 ##
##                  ## ##   ###   ### ##     ##                ##
##                 ##   ##  #### #### ##     ##                ##
##                ##     ## ## ### ## ##     ##                ##
##                ######### ##     ## ##  ## ##                ##
##                ##     ## ##     ## ##    ##                 ##
##                ##     ## ##     ##  ##### ##                ##
##                                                             ##
##                                                             ##
##                                                             ##
#################################################################

  - Stop all existing AMQ processes...

  - Red Hat JBoss AMQ Broker is present...

  - existing Red Hat JBoss AMQ Broker install detected...

  - moving existing Red Hat JBoss AMQ Broker aside...

  - Unpacking Red Hat JBoss AMQ Broker 7.8.1

  - Making sure 'AMQ' for server is executable...

  - Create Replicated Master

Creating ActiveMQ Artemis instance at: /Users/hoge/amq/target/amq-broker-7.8.1/instances/replicatedMaster

You can now start the broker by executing:  

   "/Users/hoge/amq/target/amq-broker-7.8.1/instances/replicatedMaster/bin/artemis" run

Or you can run the broker in the background using:

   "/Users/hoge/amq/target/amq-broker-7.8.1/instances/replicatedMaster/bin/artemis-service" start

  - Change default configuration to avoid duplicated live broker when failingback

  - Changing default master clustering configuration

  - Create Replicated Slave

Creating ActiveMQ Artemis instance at: /Users/hoge/amq/target/amq-broker-7.8.1/instances/replicatedSlave

You can now start the broker by executing:  

   "/Users/hoge/amq/target/amq-broker-7.8.1/instances/replicatedSlave/bin/artemis" run

Or you can run the broker in the background using:

   "/Users/hoge/amq/target/amq-broker-7.8.1/instances/replicatedSlave/bin/artemis-service" start

  - Change default configuration to automate failback

  - Changing default master clustering configuration

  - Start up AMQ Master in the background

Starting artemis-service
artemis-service is now running (18898)
  - Testing broker,retry when not ready
  - Start up AMQ Slave in the background

Starting artemis-service
artemis-service is now running (19023)
  - Testing broker,retry when not ready
  - Create haQueue on master broker

Connection brokerURL = tcp://localhost:61616
Queue [name=haQueue, address=haQueue, routingType=ANYCAST, durable=true, purgeOnNoConsumers=false, autoCreateAddress=false, exclusive=false, lastValue=false, lastValueKey=null, nonDestructive=false, consumersBeforeDispatch=0, delayBeforeDispatch=-1, autoCreateAddress=false] created successfully.

To stop the backgroud AMQ broker processes, please go to bin folders and execute 'artemis-service stop'
```


## マスタブローカーの設定ファイル

/Users/hoge/amq/target/amq-broker-7.8.1/instances/replicatedMaster/etc/broker.xml

ポート関連

|  コンポーネント  | リスナーポート  |
| ---- | ---- |
|  web  |  http://localhost:8161  |
|  artemis  |  tcp://127.0.0.1:61616  |
|  amqp  |  tcp://127.0.0.1:5672  |
|  stomp  |  tcp://127.0.0.1:61613  |
|  hornetq  |  tcp://127.0.0.1:5445  |
|  mqtt  |  tcp://127.0.0.1:1883  |



## スレーブブローカーの設定ファイル

/Users/hoge/amq/target/amq-broker-7.8.1/instances/replicatedSlave/etc/broker.xml

ポート関連　※オフセット100

|  コンポーネント  | リスナーポート  |
| ---- | ---- |
|  web  |  http://localhost:8261  |
|  artemis  |  tcp://127.0.0.1:61716  |
|  amqp  |  tcp://127.0.0.1:5772  |
|  stomp  |  tcp://127.0.0.1:61713  |
|  hornetq  |  tcp://127.0.0.1:5545  |
|  mqtt  |  tcp://127.0.0.1:1983  |


## テスト

### 1. メッセージの送信

マスターブローカーにメッセージを送信する。

```bash
$ target/amq-broker-7.8.1/instances/replicatedMaster/bin/artemis producer --message-count 10 --url "tcp://127.0.0.1:61616" --destination queue://haQueue
Connection brokerURL = tcp://127.0.0.1:61616
Producer ActiveMQQueue[haQueue], thread=0 Started to calculate elapsed time ...

Producer ActiveMQQueue[haQueue], thread=0 Produced: 10 messages
Producer ActiveMQQueue[haQueue], thread=0 Elapsed time in second : 0 s
Producer ActiveMQQueue[haQueue], thread=0 Elapsed time in milli second : 51 milli seconds
```

WEBコンソールでメッセージが10件来たことが確認出来る。

http://localhost:8161

## 2. マスターブローカーのシャットダウン

```bash
$ target/amq-broker-7.8.1/instances/replicatedMaster/bin/artemis-service stop
Gracefully Stopping artemis-service
```


## 3. スレーブブローカーでメッセージを確認する。

http://localhost:8261


## 4. フェイルバック

マスターを再起動する。

```bash
$ target/amq-broker-7.8.1/instances/replicatedMaster/bin/artemis-service start
Starting artemis-service
artemis-service is now running (25146)
```

http://localhost:8161/


## 5. マスターブローカーからメッセージを受信する。

以下のコマンドを実行する。

```bash
hoge@hoge-mac amq % target/amq-broker-7.8.1/instances/replicatedMaster/bin/artemis consumer --message-count 10 --url "tcp://127.0.0.1:61616" --destination queue://haQueue
Connection brokerURL = tcp://127.0.0.1:61616
Consumer:: filter = null
Consumer ActiveMQQueue[haQueue], thread=0 wait until 10 messages are consumed
Consumer ActiveMQQueue[haQueue], thread=0 Consumed: 10 messages
Consumer ActiveMQQueue[haQueue], thread=0 Elapsed time in second : 0 s
Consumer ActiveMQQueue[haQueue], thread=0 Elapsed time in milli second : 14 milli seconds
Consumer ActiveMQQueue[haQueue], thread=0 Consumed: 10 messages
Consumer ActiveMQQueue[haQueue], thread=0 Consumer thread finished
```

## 6. マスターブローカが復旧してもスレーブブローカーにデータが残っているので、消す必要がある。

コンシューマーでメッセージを受信する。

```bash
$ target/amq-broker-7.8.1/instances/replicatedSlave/bin/artemis consumer --message-count 10 --url "tcp://127.0.0.1:61716" --destination queue://haQueue
Connection brokerURL = tcp://127.0.0.1:61716
Consumer:: filter = null
Consumer ActiveMQQueue[haQueue], thread=0 wait until 10 messages are consumed
Consumer ActiveMQQueue[haQueue], thread=0 Consumed: 10 messages
Consumer ActiveMQQueue[haQueue], thread=0 Elapsed time in second : 0 s
Consumer ActiveMQQueue[haQueue], thread=0 Elapsed time in milli second : 13 milli seconds
Consumer ActiveMQQueue[haQueue], thread=0 Consumed: 10 messages
Consumer ActiveMQQueue[haQueue], thread=0 Consumer thread finished
hoge@hoge-mac amq % 
```

## 7. 後処理

AMQ Brokerのマスターブローカーとスレーブブローカーを停止する。

```bash
target/amq-broker-7.8.1/instances/replicatedMaster/bin/artemis-service stop
target/amq-broker-7.8.1/instances/replicatedSlave/bin/artemis-service stop
```

## 8. まとめ

マスターブローカーのbroker.xmlの設定内容

```bash
      <connectors>
         <!-- Connector used to be announced through cluster connections and notifications -->
         <connector name="artemis">tcp://127.0.0.1:61616</connector>
         <connector name="discovery-connector">tcp://127.0.0.1:61716</connector>
      </connectors>

      <cluster-user>clusterUser</cluster-user>
      <cluster-password>clusterPassword</cluster-password>
      <cluster-connections>
         <cluster-connection name="my-cluster">
            <connector-ref>artemis</connector-ref>
            <message-load-balancing>ON_DEMAND</message-load-balancing>
            <max-hops>1</max-hops>
            <static-connectors>
               <connector-ref>discovery-connector</connector-ref>
            </static-connectors>
         </cluster-connection>
      </cluster-connections>

      <ha-policy>
         <replication>
            <master>
               <vote-on-replication-failure>true</vote-on-replication-failure>
            </master>
         </replication>
      </ha-policy>
```

スレーブブローカーのbroker.xmlの設定内容

```bash
      <connectors>
         <!-- Connector used to be announced through cluster connections and notifications -->
         <connector name="artemis">tcp://127.0.0.1:61716</connector>
         <connector name="discovery-connector">tcp://127.0.0.1:61616</connector>
      </connectors>

      <cluster-user>clusterUser</cluster-user>
      <cluster-password>clusterPassword</cluster-password>
      <cluster-connections>
         <cluster-connection name="my-cluster">
            <connector-ref>artemis</connector-ref>
            <message-load-balancing>ON_DEMAND</message-load-balancing>
            <max-hops>1</max-hops>
            <static-connectors>
               <connector-ref>discovery-connector</connector-ref>
            </static-connectors>
         </cluster-connection>
      </cluster-connections>

      <ha-policy>
         <replication>
            <slave>
               <allow-failback>true</allow-failback>
            </slave>
         </replication>
      </ha-policy>
```

参考URL

https://github.com/jbossdemocentral/amq-ha-replicated-demo