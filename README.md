# Kafka Cluster using Ansible.
You can provisioned Zookeeper and Kafka cluster and get it up and running.

### Running the playbook
Ensure you have the following information correctly placed with in ansible configuration and host files.

```
bastion:~/project/ansible/ansible-kafka-cluster$ cat hosts 
[zookeepers]
10.1.2.5 zookeeper_myid=1 ansible_become=yes
10.1.2.6 zookeeper_myid=2 ansible_become=yes

[kafka-nodes]
10.1.2.8 kafka_broker_id=1 kafka_hostname=kafka-1 ansible_become=yes
10.1.2.9 kafka_broker_id=2 kafka_hostname=kafka-2 ansible_become=yes
```

```
bastion:~/project/ansible/ansible-kafka-cluster$ cat site.yml 
---
- hosts: zookeeper
  roles:
    - {
        role: hostfile,
      }
    - {
        role: java,
      }
    - {
        role: zookeeper,
        zookeeper_version: 3.6.3
      }
- hosts: kafka-nodes
  roles:
    - {
        role: hostfile,
      }
    - {
        role: java,
      }
    - {
        role: kafka
      }

```
## Testing Kafka setup.

On set of VM’s that would be used as kafka-cluster, ```'/etc/hosts'``` file should list hostnames of all kafka-nodes; something like this
```
127.0.0.1       localhost
127.0.1.1       kafka-01
# List all the kafka nodes here so that each node in cluster know about every other nodes.
10.1.2.8   kafka-01 kafka-1
10.1.2.9   kafka-02 kafka-2
```
Similarly do for kafka-02. As per example above, the ```/etc/hostname``` file on 10.1.2.8 and 10.1.2.9 nodes need to contain:
```
kafka-01
kafka-02
```
entry respectively.

Under kafka role in playbook, I enabled the ```host.name```  property i.e. hostname the broker will bind to. If not set, the server will bind to all interfaces
```
host.name={{ kafka_hostname }}
```

Then create the topic using the following command:
```
node17:/usr/local/kafka$ bin/kafka-topics.sh --create --zookeeper 10.1.2.5:2181 --replication-factor 2 --partitions 2 --topic test-2
Created topic “test-2”.
```

Verify and make sure it's created:
```
kafka-01:/usr/local/kafka$ bin/kafka-topics.sh --describe --zookeeper 10.1.2.5:2181 --topic test-2
Topic:test-2      PartitionCount:2  ReplicationFactor:2     Configs:
      Topic: test-2     Partition: 0      Leader: 2   Replicas: 2,1     Isr: 2,1
      Topic: test-2     Partition: 1      Leader: 1   Replicas: 1,2     Isr: 1,2
```

Then test this using the following commad on kafka-01 and kafka-02 respectively (I.e. Producer on one node and Consumer on another)
```
kafka-01:/usr/local/kafka$ bin/kafka-console-producer.sh --broker-list kafka-1:9092,kafka-2:9092 --sync --topic test-2
kafka-02:/usr/local/kafka$ bin/kafka-console-consumer.sh --zookeeper 10.1.2.5:2181 --from-beginning --topic test-2
```

Testing, stop the ```kafka-02``` and you can see the leaders for the defined topic has been changed accordingly.
```
kafka-01:/usr/local/kafka$ bin/kafka-topics.sh --describe --zookeeper 10.1.2.5:2181 --topic test-2
Topic:test-2      PartitionCount:2  ReplicationFactor:2     Configs:
      Topic: test-2     Partition: 0      Leader: 1   Replicas: 2,1     Isr: 1
      Topic: test-2     Partition: 1      Leader: 1   Replicas: 1,2     Isr: 1
```


