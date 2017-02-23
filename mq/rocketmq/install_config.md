
```

```

启动nameServer(172.17.32.127,172.17.32.128)

```
bash mqnamesrv & 
```


设置好ROCKETMQ_HOME
```
export ROCKETMQ_HOME=/data/RocketMQ/devenv
```

启动一个2Master+2Slave模式集群，异步复制

```
### 先启动 Name Server，例如机器 IP 为：192.168.1.1:9876
nohup sh mqnamesrv &
### 在机器 A，启动第一个 Master
nohup bash bin/mqbroker -n 172.17.32.127:9876;172.17.32.128:9876 -c $ROCKETMQ_HOME/conf/2m-2s-async/broker-a.properties &
### 在机器 B，启动第二个 Master
nohup bash bin/mqbroker -n 172.17.32.127:9876;172.17.32.128:9876 -c $ROCKETMQ_HOME/conf/2m-2s-async/broker-b.properties  &
### 在机器 C，启动第一个 Slave
nohup bash bin/mqbroker -n 172.17.32.127:9876;172.17.32.128:9876 -c $ROCKETMQ_HOME/conf/2m-2s-async/broker-a-s.properties &
### 在机器 D，启动第二个 Slave
nohup bash bin/mqbroker -n 172.17.32.127:9876;172.17.32.128:9876 -c $ROCKETMQ_HOME/conf/2m-2s-async/broker-b-s.properties &
```

