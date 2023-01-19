# Kafka service

## Starting kafka:
To start Kafka, here is the list of the command needed to run in the commad line:

To start the zookeeper, I ran the following commands:
`````
cd C:/Tools/kafka
start bin\windows\zookeeper-server-start.bat config/zookeeper.properties
`````
To start the kafka server, I ran the following command:
````
start bin\windows\kafka-server-start.bat config/server.properties
````

To test that everything is good and working properly, I started a kafka consumer and a kafka producer:
To do so, I executed the following command:
````
start bin\windows\kafka-console-consumer.bat --bootstrap-server localhost:9092 --topic R1

start bin\windows\kafka-console-producer.bat --broker-list localhost:9092 --topic R1
````
![image](https://user-images.githubusercontent.com/84817425/212467064-0edf5b0c-4ff7-4a3e-bbea-64df3538bbff.png)

## Spring cloud streams functions :

**1. Create a kafka producer via a Rest Controller:**
After creating the **`PageEventRestController.java`**, I added the following code:
```Java
 @Autowired
    private StreamBridge streamBridge;
    @GetMapping("/publish/{topic}/{name}")
    public PageEvent publish(@PathVariable String topic, @PathVariable String name){
        PageEvent pageEvent=new PageEvent(name,Math.random()>0.5?"U1":"U2",new Date(), new Random().nextInt(9000));
        streamBridge.send(topic,
                pageEvent);
        return pageEvent;
    }
```
To test this, I typed: <http://localhost:8080/publish/R1/test> on my browser to send a pageEvent object to the topic **`R1`**
This resulted in the following:
    
**On your browser:**

![image](https://user-images.githubusercontent.com/84817425/213519385-a7ead063-a909-4c35-aaa9-077d1409683d.png)

**In kafka console consumer for the topic **`R1`****

![image](https://user-images.githubusercontent.com/84817425/213519287-e927aa24-bded-40ad-a903-2b171c160ab4.png)

---
**2. Create a kafka consumer service:**
- After creating **`PageEventService.java`**, I added the following code:

```Java
@Bean
    public Consumer<PageEvent> pageEventConsumer(){
        return (input)->{
            System.out.println("***************");
            System.out.println(input.toString());
            System.out.println("***************");
        };
    }
```
- In **`application.properties`**, I added the following configuration that sets the destination for receiving messages:
```
spring.cloud.stream.bindings.pageEventConsumer-in-0.destination=R1
spring.cloud.function.definition=pageEventConsumer;
```
To test this, I typed: <http://localhost:8080/publish/R1/test> on my browser to send a pageEvent object to the topic **`R1`**
This resulted in the following:

![image](https://user-images.githubusercontent.com/84817425/213519637-f22a9f57-5d8e-4e5a-8ffc-8b1cfcc1b8aa.png)

---
**3. Create a kafka supplier service:**
- In **`PageEventService.java`**, I added the following code:

```Java
@Bean
    public Supplier<PageEvent> pageEventSupplier(){
        return()-> new PageEvent(Math.random()>0.5?"P1":"P2",
                Math.random()>0.5?"U1":"U2",
                new Date(),
                new Random().nextInt(9000));
    }
```
This method defines a Spring Framework Bean named "pageEventSupplier" that returns a Supplier<PageEvent> object. The supplier, when invoked, creates and returns a new PageEvent object with randomly generated values for its parameters: pageName, userName, date, and randomNumber. The pageName and userName are randomly determined to be either "P1" or "P2" and "U1" or "U2" respectively. The date is set to the current date when the PageEvent object is created, and the random number is generated using the nextInt() method of the Random class with a maximum value of 9000.

- In **`application.properties`**, I added the following configuration that sets the destination for sending messages:
 
```
spring.cloud.function.definition=pageEventConsumer;pageEventSupplier;
spring.cloud.stream.bindings.pageEventSupplier-out-0.destination=R2
```
Once the application is running, this method is invoked, which means opening a kafka consumer fo the topic **`R2`** will result in:
 
![image](https://user-images.githubusercontent.com/84817425/213519778-7ba1991d-6b0b-4176-907c-2924632b3e3b.png)

--- 
**4. Create a function that count and renders the number of pages depending on the key value:**
- In **`PageEventService.java`**, I added the following code:
 
```Java
@Bean
    public  Function<KStream<String,PageEvent>,KStream<String,Long>> kStreamFunction(){
        return(input)->{
            return input.filter((k,v)->v.getDuration()>100)
                    .map((k,v)->new KeyValue<>(v.getName(),0L))
                    .groupBy((k,v)->k,Grouped.with(Serdes.String(), Serdes.Long()))
                    .count()
                    .toStream();
        };
    }
```
- In **`application.properties`**, i added the following configuration that sets the destinations of two channels, one for receiving messages and the other one for sending messages respectively:
 
```
spring.cloud.function.definition=pageEventConsumer;pageEventSupplier;kStreamFunction 
spring.cloud.stream.bindings.kStreamFunction-in-0.destination=R2
spring.cloud.stream.bindings.kStreamFunction-out-0.destination=R4
```
- In the command line, I executed:
```
> start bin\windows\kafka-console-consumer.bat --bootstrap-server localhost:9092 --topic R2 
> start bin\windows\kafka-console-consumer.bat --bootstrap-server localhost:9092 --topic R4 --property print.key=true --property print.value=true --property key.deserializer=org.apache.kafka.common.serialization.StringDeserializer --property value.deserializer=org.apache.kafka.common.serialization.LongDeserializer
```
This resulted in :
 
![image](https://user-images.githubusercontent.com/84817425/213468461-738a3ac1-faae-438b-8d7f-8983551d88cb.png)
 
---
**5. Create a Data Analytics Real Time Stream Processing service with Kafka Streams:**
- In **`PageEventRestController.java`**, I added the following method : 
 ```Java
     @GetMapping(path = "/analytics", produces = MediaType.TEXT_EVENT_STREAM_VALUE)
    public Flux<Map<String, Long>> analytics(){
        return Flux.interval(Duration.ofSeconds(1))
                .map(sequence->{
                Map<String,Long> stringLongMap=new HashMap<>();
                    ReadOnlyWindowStore<String,Long> stats= interactiveQueryService.getQueryableStore("page-count", QueryableStoreTypes.windowStore());
                    Instant now = Instant.now();
                    Instant from = now.minusMillis(5000);
                    KeyValueIterator<Windowed<String>,Long> fetchAll= stats.fetchAll(from, now);
                    while (fetchAll.hasNext()){
                        KeyValue<Windowed<String>,Long> next=fetchAll.next();
                        stringLongMap.put(next.key.key(), next.value);
                    }
                    return stringLongMap;
                }).share();
    }
 ```
- To test this, I typed <http://localhost:8080/analytics> on my browser:
 
![image](https://user-images.githubusercontent.com/84817425/213541389-cc3b9693-1edc-4a2c-baab-313e95a125ea.png)

 **6- Create an html file to display Stream Data Analytics results in real time**:
 To do so, I installed smoothie.js:
 ```shell
 npm install smoothie
 ```
 then, after creating **`index.html`**, I added to it the following code:
 ```html
 <!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <title>Analytics</title>
    <script src="https://cdnjs.cloudflare.com/ajax/libs/smoothie/1.34.0/smoothie.min.js"></script>
</head>
<body>
<canvas id="chart" width="600" height="400"></canvas>
<script>
    var index = -1;
    randomColor = function () {
        ++index;
        if (index >= colors.length) index = 0;
        return colors[index];
    }
    var pages = ["P1", "P2"];
    var colors = [
        {sroke: 'rgba(0, 255, 0, 1)', fill: 'rgba(0, 255, 0, 0.2)'},
        {sroke: 'rgba(255, 0, 0, 1)', fill: 'rgba(255, 0, 0, 0.2)'
        }];
    var courbe = [];
    var smoothieChart = new SmoothieChart({tooltip: true});
    smoothieChart.streamTo(document.getElementById("chart"), 500);
    pages.forEach(function (v) {
        courbe[v] = new TimeSeries();
        col = randomColor();
        smoothieChart.addTimeSeries(courbe[v], {strokeStyle: col.sroke, fillStyle: col.fill, lineWidth: 2});
    });
    var stockEventSource = new EventSource("/analytics");
    stockEventSource.addEventListener("message", function (event) {
        pages.forEach(function (v) {
            val = JSON.parse(event.data)[v];
            courbe[v].append(new Date().getTime(), val);
        });
    });
</script>
</body>
</html>
 ```
 This resuled in: 
 
 ![image](https://user-images.githubusercontent.com/84817425/213542350-ed408f70-17cf-4ba7-b01e-eb8c706aad54.png)

 
