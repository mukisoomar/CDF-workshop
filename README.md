# CDF Labs: Real-time Web Click Stream Data analysis with NiFi, Kafka, Druid, Zeppelin and Superset


## Prerequisite

**Although this AMI is not public and is available for Cloudera workhops only, the steps can be reproduced in your own environment**

- Launch AWS AMI **ami-06c4071a1b5d62c4f** with **m5d.4xlarge** instance type
- Keep default storage (300GB SSD)
- Set security group with:
  - Type: All TCP
  - Source: My IP
- Choose an existing or create a new key pair

## Content

* [Lab 1 - Accessing the sandbox](#accessing-the-sandbox)
* [Lab 2 - Stream data using NiFi](#stream-data-using-nifi)
* [Lab 2.1 - Build the first NiFi flow](#Build-the-first-NiFi-flow)
* [Lab 2.2 - Process and enrich content](#Process-and-enrich-content)
* [Lab 3 - Explore Kafka](#explore-kafka)
* [Lab 4 - Integrate with Schema Registry](#integrate-with-schema-registry)
* [Lab 5 - Explore Hive, Druid and Zeppelin](#explore-hive-druid-and-zeppelin)
* [Lab 6 - Stream enhanced data into Hive using NiFi](#stream-enhanced-data-into-hive-using-nifi)
* [Lab 7 - Create live dashboard with Superset](#create-live-dashboard-with-superset)
* [Lab 8 - Collect syslog data using MiNiFi and EFM](#collect-syslog-data-using-minifi-and-efm)
* [Bonus - Process sentiment analysis on tweets](#process-sentiment-analysis-on-tweets)

## Accessing the sandbox

### Add an alias to your hosts file

On Mac OS X, open a terminal and vi /etc/hosts

On Windows, open C:\Windows\System32\drivers\etc\hosts

Add a new line to the existing

```nnn.nnn.nnn.nnn	demo.cloudera.com```

Replacing the ip (nn.nnn.nnn.nnn) address with the one provided

If you can't edit the hosts file due to lack of privileges, then you will need to replace the reference to demo.cloudera.com alias with the instance private ip wherever it's used by a NiFi processor.

To get this private ip, ssh to the instance and type the command ```ifconfig```, the first ip starting with 172 is the one to use:

![Private IP](images/private-ip.png)

### Start all HDP and CDF services

**They should be already started**

Open a web browser and go to the following url

```http://demo.cloudera.com:8080/```

Log in with the following credential

Username: admin
Password: admin

If services are not started already, start all services

![Image of Ambari Start Services](images/start_services.png)

It can take up to 20 minutes...

![Image of Ambari Starting Services](images/starting_services.png)

### SSH to the sandbox

**Copy and paste** (do not download) the content of [ppk](keys/WI-HDF-WORKSHOP-KEYS.ppk) for Windows or [pem](keys/WI-HDF-WORKSHOP-KEYS.pem) for Mac OS X

On Mac use the terminal to SSH

For Mac users, don't forget to ```chmod 400 /path/to/hdp-workshop.pem``` before ssh'ing

![Image of Mac terminal ssh](images/mac_terminal_ssh.png)

On Windows use [putty](https://www.chiark.greenend.org.uk/~sgtatham/putty/latest.html)

![Image of Putty ssh](images/login_with_putty_1.png)

![Image of Putty ssh](images/login_with_putty_2.png)

## Stream data using NiFi
In these labs while exploring the Cloudera Platform technologies, we are going to build a Web Click Stream Analytics Application that will allow us to monitor the products that our customers or web site visitors are most interested in. For this we are going to assume that we are a financial institution that exposes its financial products to its customers as well as general public and will monitor the interest in the products advertised through our website.

While this example is for a financial institution, this use case in general is applicable in any vertical including retail and other consumer oriented businesses. The monitoring aspect of the interest in the products could generally be for understanding the consumer behavior for the products or also for understanding the efficacy of a marketing campaign for new products or for promotions of existing products through a website. The insights derived from real-time monitoring and notifications can be leveraged for making decisions in real-time on which products to focus the most or which segment of the consumers to target the most. Analysis of many such scenarios become feasible when you have the right information at the right time, particularly in real-time so that you can take pro-active actions in real-time. 

In order to have a web-click stream data source available for our workshop, we are going to make use of a script that will simulate the web application and generate web-click stream data. We will capture that data via **NiFi**, filter appropriate events for routing purposes, process it where required, and forward it to downstream applications via **Kafka** for further analysis. **NiFi and Kafka**, together make streaming analytics possible as you will see by working through the labs. 

**TODO: Image of the overall architecture.**

****
### Build the first NiFi flow

Let's get started... Open [NiFi UI](http://demo.cloudera.com:9090/nifi/) and follow the steps below:


- **Step 1:** Drag on drop a Process Group on the root canvas and name it **CDF Workshop**

![CDF Workshop process group](images/cdfprocessgroup.png)

- **Step 2:** Go to [NiFi Registry](http://demo.cloudera.com:61080/nifi-registry/explorer/grid-list) and create a new bucket
  - Click on the little wrench icon at the top right corner
  - Click on the **NEW BUCKET** button
  - Name the bucket **workshop**
  
![NiFi Registry bucket creation](images/registry-bucket.png)

- **Step 3:** Go back to NiFi UI and right click on the previously created process group
  - Click on Version > Start version control
  - Then provide at least a Flow Name - clickstream-flow
  - Click on Save
  
  This will put your process group in version control. As you build your flows by adding processors, you can persist them from time to time in the flow registry for version control.

![Version control](images/version-control.png.png)

![Version control 2](images/version-control-2.png.png)


- **Step 4: Explore the web-app simulator script**  

   We will now gradually build the process flow to capture web logs. Since we dont have a real web application, we are going to use a web-application simulator that will generate the log files. Let us explore how this simulator works first. Follow the below steps:  
  - SSH into your instance.
  - cd to `/home/centos/cdf-workshop-master/data_gen Directory`
  - There are  3 scripts here as shown below:
	![Version control 2](images/data_gen.1.png.png)
  - `generate-clickstream-data.sh`: This is the main script. It reads the data from the `/home/centos/cdf-workshop-master/data/filtered-omniture-raw.tsv` file. This file is a real log data fiel from a web application. It uses this file but replaces the urls with randomly generated urls that are provided in the `products.tsv` file. You can change the list of urls here to reflect any product you want simulated. For our labs, we are simulating these web logs are for a financial institution that have financial products. Open up the two files and explore the data content in them.  
  Execute the  `generate-clickstream-data.sh` script and see what kind of data is generated by typing `.\generate-clickstream-data.sh`.
  - `publish-clickstream-to-nifi.sh` : This script uses the `generate-clickstream-data.sh` to publish the same data to a tcp socket. You can control the speed at which this data is generated by providing arguments like `0.5, 1.0m,etc`. The 's', 'm' are the units for seconds and minutes. with no units, the number will be treated as seconds. For our initial data flow in NiFi, **we are going to use this script to publish to a NiFi TCPListen processor.** You can execute this script and see what it does. 
  - `write-clickstream-to-file.sh` : This script writes the output to a file in the "../data/weblogs" directory in a file called "weblogs.log". **We will use this file later to have minifi capture the logs written to this file and publish it over to NiFi.**  


- **Step 5: Configure a ListenTCP Processor**  
 
  Get into the CDF Workshop process group (double click on the process group).  
  - Drag a **ListenTCP** processor to the canvas
  - Double click on the processor
  - On settings tab, change the **Name** parameter value to **Listen for clickstream log events**
  - On properties tab, change **Port** value to **9797**
  - You can hover on the **?** icon by each property value to see what they are for, but leave the default values as they are.
  - Apply changes
  ![listenTCP properties](images/TCP-Listener-Config-1.png.png)
  - Drag the **funnel icon** from the menu bar at the top on the canvas.
  - Connect the **ListenTCP** processor to the **funnel icon**. A **Create Connection** icon pops up with the **success** box checked. Accept the default and click on the **Add button*
  - The **funnel** is used to typically to collect flowfiles from different connections. However we are using it here as a place-holder for future flows and to collect the data that comes in from the **ListenTCP Processor** and send that data to the intermediate queue to help explore the data as it comes in.
  ![listenTCP processor flow](images/TCP-Listener-Config-2.png.png)
  
   - Start the processor by righ-clicking on the processor and clicking on the **start** menu item. This will start the processor and it will now listen on the port **9797** for incoming packets.
   - Go back to your command prompt window and now execute the publish-clickstream-to-nifi.sh script by executing the following command.
    `.\publish-clickstream-to-nifi.sh 1`. 
   You will now see that the script executes and does not error out since it is now able to connect to the **ListenTCP** processor and send the data packets over.
   - Within your NiFi flow, you should now see data coming in and the messages in the queue connection piling up.
   ![listenTCP processor consuming messages from publish-clickstream-to-nifi.sh script](images/TCP-Listener-Config-3.png.png)
   - Stop the **publish-clickstream-to-nifi.sh** script by using **CTL-C**. 

- **Step 6: Explore data in queue**  
 
  We are now going to explore what came into the ListenTCP processor and how this data is now available from the queue connection we created between the processor and the funnel.  
  - Right click on the **success queue** and from the context menu, select **List Queue** item. ![Selecting List Queue](images/Queue-list-1.png.png)
  - This will open up a window showing all the flowfiles that were received by the **ListenTCP** processor and forwarded to the connection queue. ![Selecting List Queue](images/Queue-list-2.png.png)
  - Select the "info icon in the first column", This will open up the a window to show the corresponding flow file details. Observe some of the attributes on the **DETAILS** tab. Each flowfile has a unique id associated with that and a unique filename given to it. Also shows the size of the file. You can download the contents of the file to your computer by clicking on the **Download Button** or click on the **View Button** to view what was received. ![Flowfile Details](images/Queue-list-3.png.png)
  - Click on the **View Button** and you will see the contents in another tab of your browser window that pops up. Keep this window open for using later. ![Flowfile Contents](images/Queue-list-4.png.png)
  - Go back to your **FlowFile** details window. Click on the **ATTRIBUTES** tab. This provides the details of the attributes that are associated with the flow file. Click OK and close the queue list window to return back to your canvas. ![Flowfile Contents](images/Queue-list-5.png.png)

****  
### Process and enrich content
In this lab, we will further enrich and process content that was received from the log generator simulating a web click stream.

We will perform the following steps to continue to build our flow:


- **Step 1: Configure the UpdateAttribute NiFi Processor**  
To reference the name of the schema in the registry that will be used for processing content within our flows, we will configure an UpdateAttribute processor.  
  - Drag an UpdateProcessor to the canvas.
  - On the **SETTINGS** tab, change the Name to "*Set Schema Name from Registry*"
  - On the **PROPERTIES** tab, click on the "+" button on the top-right side of the window and an attribute. Set the values as follows:    
    - Property Name: schema.name
    - Property Value: clickstream_event 
  - Click OK, APPLY and close the processor properties.
  - Connect the "*Listen for clickstream logs*" processor to this processor, using the "*success*" relationship. A connection queue will show up on the connection line joining the two processors.![UpdateProcessor-Properties-2](images/SetSchemaNamefromRegistry_1.png.png)
  ![UpdateProcessor-Properties-2](images/SetSchemaNamefromRegistry_2.png.png)

  
 - **Step 2: Define clickstream_events schema and register with Schema Registry**  
To be able to parse the data received from the clickstream log events, we will need to defined a data structure that can be referenced by various services to parse or serialize and de-serialize the data when required.  For this we will define a schema called **clicstream_event** and persist into the schema registry.   


   Explore [Schema Registry UI](http://demo.cloudera.com:7788/)   

  Create a new Avro Schema, hitting the plus button, named **clickstream_event** with the following Avro Schema. You an copy and paste the schema in the **SCHEMA TEXT** window. Fill in the other values as shown in the figure below:   

```
{
 "type": "record",
 "namespace": "cloudera.workshop.clickstream",
 "name": "clickstream_event",
 "fields": [
  {
   "name": "clickstream_id",
   "type": "string"
  },
  {
   "name": "timestamp",
   "type": [
    "null",
    "string"
   ]
  },
  {
   "name": "IPaddress",
   "type": [
    "null",
    "string"
   ]
  },
  {
   "name": "url",
   "type": [
    "null",
    "string"
   ]
  },
  {
   "name": "is_purchased",
   "type": [
    "null",
    "string"
   ]
  },
  {
   "name": "is_page_errored",
   "type": [
    "null",
    "string"
   ]
  },
  {
   "name": "user_session_id",
   "type": [
    "null",
    "string"
   ]
  },
  {
   "name": "city",
   "type": [
    "null",
    "string"
   ]
  },
  {
   "name": "state",
   "type": [
    "null",
    "string"
   ]
  },
  {
   "name": "country",
   "type": [
    "null",
    "string"
   ]
  }
 ]
}
```

![Avro schema creation](images/avro_schema_creation.png)  

  You should end up with a newly versioned schema as follow:

![Avro schema versioned](images/avro_schema_versioned.png)

  Explore the [REST API](http://demo.cloudera.com:7788/swagger) as well. You can use these APIs to perform various actions on the schemas. 

  Additionally you can explore by clicking on the *Edit* and *Fork* the features they provide for maintaining the schemas along with publishing the new versions for general consumption by other flows or services. (*Note: Ignore the name of the schmea showing up as clickstream_event_v1 in the images.*)
  
 - **Step 3: Configure a SplitRecord Procesor**
   
In this step, we will configure a **SplitRecord** processor. There are two reasons for this. (1) to be able to split the content received into individual records. The content received over ListenTCP processor can have multiple records coming in one data flow file depending on the speed of the streaming data as well as the size of the buffer configured. (2) Convert the data format into a format required for further processing or delivering it to a destination.   

The data coming in is in the pipe delimited format. We will convert it into json format so we can extract the data we need using another processor in the next step. 

Drag the SplitRecord processor on the canvas and perform the following steps:  
    - On the **PROPETIES** tab  
      - RecordReader: Click on CreateNewService in the dropdown and select CSVReader
      - RecordWriter: Click on the CreateNewService in the dropdwon and select JsonRecordSetWriter
      - Records Per Split: 1  
      ![CSVReader Config](images/SplitRecord-CSVReader-1.png.png)
      - Click on arrow next to CSVReader. It will ask you to save the configurations, which you can accept. It will then take you to the **CONTROLLER SERVICES** window. Click on the **Gear** icon and select the **PROPERTIES** tab. Set the following properties as below:  
        - Scheama Access Strategy: *Use 'Schema Name' Property*
        - Scheama Registry: Select 'create new service' and select HortonworksSchemaRegistry
         ![HortonworksRegistry Config](images/SplitRecord-CSVReader-2.png.png)
        - Schema Name: $(schema.name)
    - On the **SETTINGS** tab  
      - Check the failure and original check boxes.  

      
Click **APPLY** and close the processor configuration window.

Connect  


For this we will first define  
- **Step 7 TODO:** Add an UpdateAttribute connector to the canvas and link from ConnectWebSocket on **text message** relationship
  - Double click on the processor
  - On properties tab add new property **mime.type** clicking on + icon and give the value **application/json**. This will tell the next processor that the messages sent by the Meetup WebSocket is in JSON format.
  - Add another property **event** to set an event name **CDF workshop** for the purpose of this exercise as explained before
  - Apply changes
  
![UpdateAtrribute 1 properties](images/updateattibute1properties.png)
  
- **Step 6:** Add EvaluateJsonPath to the canvas and link from UpdateAttribute on **success** relationship
  - Double click on the processor
  - On settings tab, check both **failure** and **unmatched** relationships
  - On properties tab, change **Destination** value to **flowfile-attribute**
  - And add properties as follow
    - comment: $.comment
    - member: $.member.member_name
    - timestamp: $.mtime
    - country: $.group.country
    
![EvaluateJsonPath 1 properties](images/evaluatejsonpathproperties1.png)
    
    The messages coming out of the web sockets look like this:
    
    ```json
    {"visibility":"public","member":{"member_id":11643711,"photo":"https:\/\/secure.meetupstatic.com\/photos\/member\/3\/1\/6\/8\/thumb_273072648.jpeg","member_name":"Loka Murphy"},"comment":"I didn’t when I registered but now thinking I want to try and get one since it’s only taking place once.","id":-259414201,"mtime":1541557753087,"event":{"event_name":"Tunnel to Viaduct 8k Run","event_id":"256109695"},"table_name":"event_comment","group":{"join_mode":"open","country":"us","city":"Seattle","name":"Seattle Green Lake Running Group","group_lon":-122.34,"id":1608555,"state":"WA","urlname":"Seattle-Greenlake-Running-Group","category":{"name":"fitness","id":9,"shortname":"fitness"},"group_photo":{"highres_link":"https:\/\/secure.meetupstatic.com\/photos\/event\/9\/e\/f\/4\/highres_465640692.jpeg","photo_link":"https:\/\/secure.meetupstatic.com\/photos\/event\/9\/e\/f\/4\/600_465640692.jpeg","photo_id":465640692,"thumb_link":"https:\/\/secure.meetupstatic.com\/photos\/event\/9\/e\/f\/4\/thumb_465640692.jpeg"},"group_lat":47.61},"in_reply_to":496130460,"status":"active"}
    ```

- Step 7: Add an AttributesToCSV processor to the canvas and link from EvaluateJsonPath on **matched** relationship
  - Double click on the processor
  - On settings tab, check **failure** relationship
  - Change **Destination** value to **flowfile-content**
  - Change **Attribute List** value to write only the above parsed attributes: **timestamp,event,member,comment,country**
  - Set **Include Schema** to **true**
  - Apply changes
  
- Step 8: Add a PutFile processor to the canvas and link from AttributesToCSV on **success** relationship
  - Double click on the processor
  - On settings tab, check all relationships
  - Change **Directory** value to **/tmp/workshop**
  - Apply changes

- Step 9: Right-click anywhere on the canvas and commit your first flow!

![NiFi commit 1](images/commit-1.png)
![NiFi commit 2](images/commit-2.png)

If you visit the [NiFi Registry UI](http://demo.cloudera.com:61080/nifi-registry/explorer/grid-list) again you should see your commit.

![NiFi commit 3](images/commit-3.png)

- Step 10: Start the entire flow

![NiFi Flow 1](images/flow1.png)

SSH to the sandbox and explore the files created under /tmp/workshop.

On the NiFi UI, explore the FlowFiles' attributes and content looking at Data provenance.

**Once done, stop the flow and delete all files ```sudo rm -rf /tmp/workshop/*```**

## Explore Kafka

ssh to the AWS instance as explained above then become root

```sudo su -```

Navigate to Kafka

```cd /usr/hdp/current/kafka-broker```

Create a topic named **meetup_comment_ws**

```./bin/kafka-topics.sh --create --zookeeper demo.cloudera.com:2181 --replication-factor 1 --partitions 1 --topic meetup_comment_ws```

List topics to check that it's been created

```./bin/kafka-topics.sh --list --zookeeper demo.cloudera.com:2181```

Open a consumer so later we can monitor and verify that JSON records will stream through this topic:

```./bin/kafka-console-consumer.sh --bootstrap-server demo.cloudera.com:6667 --topic meetup_comment_ws```

Keep this terminal open.

We will now open a new terminal to publish some messages...

Follow the same steps as above except for the last step where we are going to open a producer instead of a consumer:

```./bin/kafka-console-producer.sh --broker-list demo.cloudera.com:6667 --topic meetup_comment_ws```

Type anything and click enter. Then go back to the first terminal with the consumer running. You should see the same message get displayed!

## Integrate with Schema Registry



  
  

--- Muki TODO: -----
Remove the last processor PutFile as we are going to stream the avro record to some Kafka topic.

Now we are going to filter the records we are interested in and convert them from CSV to Avro in the process.

- Step 1: Remove the existing PutFile processor and add a QueryRecord processor to the canvas and link from AttributesToCSV on **success** relationship
  - Double click on the processor
  - On properties tab
  	- For RecordReader, create a CSVReader service
  	- For RecordWrite, create a AvroRecordSetWriter service
  	- Configure both services
  	  - For CSVReader, use String Fields From Header for the Schema Access Strategy
  	  - For AvroRecordSetWriter, we are going to connect to the Schema Registry API (http://demo.cloudera.com:7788/api/v1) and use the avro schema created before. For the **Schema Registry** property choose **HortonworksSchemaRegistry** as shown in the screen shot below and configure it.
  	    - For **Schema Access Strategy** use 'Schema Name' property
  	    - For **Schema Name** property add ```meetup_comment_avro```
  	- Filter comments per country as our sentiment analysis model supports English only
  	  - Add a property **comments_in_english** with value ```SELECT * FROM FLOWFILE WHERE country IN ('gb', 'us', 'sg')```
  	- Set **Include Zero Record FlowFiles** to false
  - On settings tab, check **failure** and **original** relationships
  - Apply changes
  
![Query Record](images/query_record.png)

![CSVReader](images/csv_reader.png)

![AvroRecordSetWriter](images/avro_record_set_writer.png)

Set the HortonworksSchemaRegistry controller service as follow

![HWXSchemaRegistry](images/hwx_schema_registry.png)

Enable all controller services.

- Step 2: Add a **PublishKafka_2_0** connector to the canvas and link from QueryRecord on **comments_in_english** relationship
  - Double click on the processor
  - On settings tab, check all relationships
  - On properties tab
  - Change **Kafka Brokers** value to **demo.cloudera.com:6667**
  - Change **Topic Name** value to **meetup_comment_ws**
  - Change **Use Transactions** value to **false**
  - Apply changes

The flow should look like this:

![Avro records to Kafka topic](images/avro_records_to_kafka_topic.png)

Again commit your changes and start the flow!

You should be able to see records streaming through Kafka looking at the terminal with Kafka consumer opened earlier

![Kafka topic avro](images/kafka_topic_avro.png)

When you are happy with the outcome stop the flow and purge the Kafka topic as we are going to use it later:

```./bin/kafka-configs.sh --zookeeper demo.cloudera.com:2181 --entity-type topics --alter --entity-name meetup_comment_ws --add-config retention.ms=1000```

Wait for few second and set the retention back to one hour:

```./bin/kafka-configs.sh --zookeeper demo.cloudera.com:2181 --entity-type topics --alter --entity-name meetup_comment_ws --add-config retention.ms=3600000```

You can check if the retention was set properly:

```./bin/kafka-configs.sh --zookeeper demo.cloudera.com:2181 --describe --entity-type topics --entity-name meetup_comment_ws```

## Explore Hive, Druid and Zeppelin

Visit [Zeppelin](http://demo.cloudera.com:9995/) and log in as admin (password: admin)

Create a new note(book) called Demo (use jdbc as default interpreter)

![Zeppelin note creation](images/zeppelin_create_note.png)

Add the interpreter to connect to Hive LLAP

```%jdbc(hive_interactive)```

Create a database named workshop and run the SQL

```SQL
CREATE DATABASE workshop;
```

Create the Hive table backed by Druid storage where the social medias sentiment analysis will be streamed into

```SQL
CREATE EXTERNAL TABLE workshop.meetup_comment_sentiment (
`__time` timestamp,
`event` string,
`member` string,
`comment` string,
`sentiment` string
)
STORED BY 'org.apache.hadoop.hive.druid.DruidStorageHandler'
TBLPROPERTIES (
"kafka.bootstrap.servers" = "demo.cloudera.com:6667",
"kafka.topic" = "meetup_comment_ws",
"druid.kafka.ingestion.useEarliestOffset" = "true",
"druid.kafka.ingestion.maxRowsInMemory" = "5",
"druid.kafka.ingestion.startDelay" = "PT1S",
"druid.kafka.ingestion.period" = "PT1S",
"druid.kafka.ingestion.consumer.retries" = "2"
);
```

![Zeppelin create DB and table](images/zeppelin_create_db_and_table.png)

Start Druid indexing

```SQL
ALTER TABLE workshop.meetup_comment_sentiment SET TBLPROPERTIES('druid.kafka.ingestion' = 'START');
```

Verify that supervisor and indexing task are running from the [Druid overload console](http://demo.cloudera.com:8090/console.html)

![Druid console](images/druid_console.png)

## Stream enhanced data into Hive using NiFi

### Run the sentiment analysis model as a REST-like service

For the purpose of this exercise we are not going to train, test and implement a classification model but re-use an existing sentiment analysis model, provided by the Stanford University as part of their [CoreNLP - Natural language software](https://stanfordnlp.github.io/CoreNLP/)

First, after ssh'ing to the sandbox, download and unzip the CoreNLP using the wget as below:

```bash
wget http://nlp.stanford.edu/software/stanford-corenlp-full-2018-10-05.zip
unzip stanford-corenlp-full-2018-10-05.zip
```

Then, in order to start the [web service](https://stanfordnlp.github.io/CoreNLP/corenlp-server.html), run the [CoreNLP jar file](https://stanfordnlp.github.io/CoreNLP/download.html), with the following commands:

```bash
cd /path/to/stanford-corenlp-full-2018-10-05
java -mx1g -cp "*" edu.stanford.nlp.pipeline.StanfordCoreNLPServer -port 9999 -timeout 15000 </dev/null &>/dev/null &
```

This will run in the background on port 9999 and you can visit the [web page](http://demo.cloudera.com:9999/) to make sure it's running.

If you want to play with it, remove all annotations and use **Sentiment** only

![Image of CoreNLP web service](images/corenlp_web.png)

The model will classify the given text into 5 categories:

- very negative
- negative
- neutral
- positive
- very positive

### Enhance the Meetup comments with sentiment analysis outcome 

Go back to [NiFi UI](http://demo.cloudera.com:9090/nifi/) and follow the steps below. Between the QueryRecord and PublishKafka_2_0 processors we are going to enhance the Meetup comment with NLP.

- Step 1: Prepare the content to be posted to the sentiment analysis service
  - Add ReplaceText processor and link from QueryRecord on **comments_in_english** relationship
  - Keep the PublishKafka_2_0 processor on the canvas unlinked from/to other processors for now
  - Double click on processor and check **failure** on settings tab
  - Go to properties tab and remove value for **Search Value** and set it to empty string
  - Set **Replacement Value** with value: **${comment:replaceAll('\\.', ';')}**. We want to make sure the entire comment is evaluated as one sentence instead of one evaluation per sentence within the same comment.
  - Set **Replacement Strategy** to **Always Replace**
  - Apply changes
  
- Step 2: Call the web service started earlier on incoming message
  - Add InvokeHTTP processor and link from ReplaceText on **success** relationship
  - Double click on processor and check all relationships except **Response** on settings tab
  - Go to properties tab and set value for **HTTP Method** to **POST**
  - Set **Remote URL** with value: ```http://demo.cloudera.com:9999/?properties=%7B%22annotators%22%3A%22sentiment%22%2C%22outputFormat%22%3A%22json%22%7D``` which is the url encoded value for **http://demo.cloudera.com:9999/?properties={"annotators":"sentiment","outputFormat":"json"}**
  - Set **Content-Type** to **application/x-www-form-urlencoded**
  - Apply changes
  
- Step 3: Add EvaluateJsonPath to the canvas and link from InvokeHTTP on **Response** relationship
  - Double click on the processor
  - On settings tab, check both **failure** and **unmatched** relationships
  - On properties tab
  - Change **Destination** value to **flowfile-attribute**
  - Add on the property **sentiment** with value **$.sentences[0].sentiment**
  - Apply changes
  
- Step 4: Format post time to comply with [ISO format](https://en.wikipedia.org/wiki/ISO_8601) (Druid requirement)
  - Add UpdateAttribute processor and link from EvaluateJsonPath on **matched** relationship
  - Using handy [NiFi's language expression](https://nifi.apache.org/docs/nifi-docs/html/expression-language-guide.html#dates), add a new attribue ```__time``` with value: ```${timestamp:format("yyyy-MM-dd'T'HH:mm:ss'Z'", "Asia/Singapore")}``` to properties tab

- Step 5: Add AttributesToJSON processor to prepare the message to be published to the Kafka topic created before. Link from UpdateAttribute.
  - Double click on processor
  - On settings tab, check **failure** relationship
  - Go to properties tab
  - In the Attributes List value set ```__time, event, comment, member, sentiment``` to match the previously created Hive table
  - Change Destination to **flowfile-content**
  - Set Include Core Attributes to **false**
  - Apply changes
  
- Step 6: Last step, link existing **PublishKafka_2_0** connector to the canvas from AttributesToJSON on **success** relationship
  
Before starting the NiFi flow make sure that the [sentiment analysis Web service](https://github.com/charlesb/CDF-workshop#run-the-sentiment-analysis-model-as-a-rest-like-service) is running
	
The overall flow should look like this

![NiFi Flow 2](images/nifi_stream_to_kafka.png)

You should be able to see records streaming through Kafka looking at the terminal with Kafka consumer opened earlier

![Kafka topic consumer](images/kafka_topic_consumer.png)

Going back to Zeppelin, we can query the data streamed in real-time

![Kafka topic consumer](images/zeppelin_monitor_sentiment_analysis.png)

## Create live dashboard with Superset

Go to [Superset UI](http://demo.cloudera.com:9088/)

Log in with user **admin** and password **admin**

Refresh Druid metadata

![Refresh druid metadata](images/superset_refresh_datasources.png)

Edit the **meetup_comment_sentiment** datasource record and verify that the columns are listed, same for the metric (you might need to scroll down)

![Druid datasource columns](images/druid_datasource_columns.png)

Click on the datasource and create the following query

![Druid query](images/druid_query.png)

From this query, create a dashboard that will refresh automatically

![Druid dashboard](images/druid_dashboard.png)

## Collect syslog data using MiNiFi and EFM

Go to NiFi Registry and create a bucket named **demo**

As root (sudo su -) start EFM and MiNiFi

```bash
service efm start
service minifi start
```
Visit [EFM UI](http://demo.cloudera.com:10080/efm/ui/)

You should see heartbeats coming from the agent

![EFM agents monitor](images/efm-agents-monitor.png)

Now, **on the root canvas**, create a simple flow to collect local syslog messages and forward them to NiFi, where the logs will be parsed, transformed into another format and pushed to a Kafka topic.

Our agent has been tagged with the class 'demo' (check nifi.c2.agent.class property in /usr/minifi/conf/bootstrap.conf) so we are going to create a template under this specific class

But first we need to add an Input Port to the root canvas of NiFi and build a flow as described before. Input Port are used to receive flow files from remote MiNiFi agents or other NiFi instances.

![NiFi syslog parser](images/nifi-syslog-parser.png)

Don't forget to create a new Kafka topic as explained in Lab 3 above.

We are going to use a Grok parser to parse the syslog messages. Here is a Grok expression that can be used to parse such logs format:

```%{SYSLOGTIMESTAMP:syslog_timestamp} %{SYSLOGHOST:syslog_hostname} %{DATA:syslog_program}(?:\[%{POSINT:syslog_pid}\])?: %{GREEDYDATA:syslog_message}```

Now that we have built the NiFi flow that will receive the logs, let's go back to the EFM UI and build the MiNiFi flow as below:

![CEM flow](images/cem-minifi-flow.png)

This MiNiFi agent will tail /var/log/messages and send the logs to a remote process group (our NiFi instance) using the Input Port.

![Tailfile](images/tail-file.png)

Don't forget to increase the scheduler!

Please note that the NiFi instance has been configured to receive data over HTTP only, not RAW

![Remote process group](images/remote-process-group.png)

Now we can start the NiFi flow and publish the MiNiFi flow to NiFi registry (Actions > Publish...)

Visit [NiFi Registry UI](http://demo.cloudera.com:61080/nifi-registry/explorer/grid-list) to make sure your flow has been published successfully.

![NiFi Registry](images/nifi-registry.png)

Within few seconds, you should be able to see syslog messages streaming through your NiFi flow and be published to the Kafka topic you have created.

![Syslog message](images/syslog-json.png)

## Process sentiment analysis on tweets

### Apply for Twitter developer account and create an app

Visit [Twitter developer page](https://developer.twitter.com/en/apply-for-access.html) and click on Apply for a developer account. If you don't have a Twitter account, sign up.

After you have added a valid email address, follow the different account creation steps

![twitter-account-details](images/twitter-account-details.png)

![twitter-usecase-details](images/twitter-usecase-details.png)

For the use case details, you can reuse the text below:

```
1. This account will be used for demo, building streaming data flow with real-time sentiment analyses using NiFi, Kafka and Druid
2. I intend to compare tweets against a machine learning model using NLP techniques for sentiment analysis
3. My use case does not involve tweeting, retweeting or liking content
4. Individual tweets will not be displayed, data will be aggregated per sentiment: very negative, negative, neutral, positive and very positive
```

Agree to the Terms of Services

![twitter-termsofservices-details](images/twitter-termsofservices-details.png)

Finally click on the link from the email you have received and create an app

![twitter-createanapp](images/twitter-createanapp.png)

![twitter-appdetails](images/twitter-appdetails.png)

Finally create the Keys and Tokens that will be needed by the NiFi processor to pull Tweets using the Twitter API

![twitter-keysandtokens](images/twitter-keysandtokens.png)

## Create a NiFi flow

- Step 1: Get the tweets to be analysed
  - Add GetTwitter processor to the canvas
  - **Important!** From Scheduling tab change Run Schedule property to **2 sec**
  - On the Properties tab
    - Set Twitter Endpoint to **Filter Endpoint**
    - Fill the Consumer Key, Consumer Secret, Access Token and Access Token Secret with the app keys and token
    - Set Languages to **en**
    - Provide a set of Terms to Filter On, i.e. 'cloudera'
  - Apply changes
  
- Step 2: Call the web service started earlier on incoming message
  - Add InvokeHTTP processor and link from ReplaceText on **success** relationship
  - Double click on processor and check all relationships except **Response** on settings tab
  - Go to properties tab and set value for **HTTP Method** to **POST**
  - Set **Remote URL** with value: ```http://demo.cloudera.com:9999/?properties=%7B%22annotators%22%3A%22sentiment%22%2C%22outputFormat%22%3A%22json%22%7D``` which is the url encoded value for **http://demo.cloudera.com:9999/?properties={"annotators":"sentiment","outputFormat":"json"}**
  - Set **Content-Type** to **application/x-www-form-urlencoded**
  - Apply changes












