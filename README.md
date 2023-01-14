# Kafka service
To start Kafka, here is the list of the command you need to run in your commad line:

To start the zookeeper, run the following commands:
`````
cd C:/Tools/kafka
start bin\windows\zookeeper-server-start.bat config/zookeeper.properties
`````
To start the kafka server, run the following command:
````
start bin\windows\kafka-server-start.bat config/server.properties
````

To test that everything is good and working properly, you need to start a kafka consumer and a kafka producer:
To do so, execute the following command:
````
start bin\windows\kafka-console-consumer.bat --bootstrap-server localhost:9092 --topic R1

start bin\windows\kafka-console-producer.bat --broker-list localhost:9092 --topic R1
````