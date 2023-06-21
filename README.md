## 2.2. Solution 2 : OBS -> DLI -> DMS Kakfa -> Logstash -> CSS

### 2.2.1 - OBS

- Enable **OBS Logging** feature on OBS Bucket **nhe-obs-eu-w0-demo**
- Record logs to 
	- Bucket : **nhe-obs-eu-w0-logs**
	- Directory : **nhe-obs-eu-w0-demo-log/**
- Reference OBS Logging: https://docs.prod-cloud-ocb.orange-business.com/usermanual/obs/en-us_topic_0045853553.html

![[Pasted image 20230619142422.png]]

### 2.2.2 - DMS Kafka

Flexible Engine DMS (Data Management Service) Kafka is a powerful and flexible cloud-based data streaming service provided by Flexible Engine, designed to simplify the management and processing of high-volume, real-time data streams. As a distributed streaming platform, Kafka enables you to build scalable, fault-tolerant, and highly available applications that can handle massive data ingestion, processing, and analysis.

Kafka acts as a robust and efficient middle layer between data producers and consumers, allowing you to seamlessly publish, subscribe, and process streaming data in real-time. With its distributed architecture and built-in fault tolerance, Kafka ensures data reliability and fault recovery even in the face of hardware failures or network outages. This makes it an ideal solution for mission-critical applications where data integrity and availability are paramount.

One of the key features that make Flexible Engine DMS Kafka stand out is its ability to handle high-throughput, low-latency data streaming. It can efficiently process and distribute millions of messages per second, making it an excellent choice for use cases such as real-time analytics, event-driven architectures, log aggregation, and data integration.

#### 2.2.2.1 Create DMS Queue 

- Create DMS Kafka Queue named **nhe-dms-kafka-demo**
- Make sure to use the VPC where the others resources could be created as well : VPC (**nhe-vpc-demos**) and Subnet (**subnet-7096-dms-demo**)

![[Pasted image 20230619143647.png]]

![[Pasted image 20230619143520.png]]

- The DMS Kafka queue should be **running** 
 
![[Pasted image 20230619143913.png]]

#### 2.2.2.2 Create DMS Topics

- Create DMS Topics **obsaccesslogs** with Flushing options

 ![[Pasted image 20230619155516.png]]

### 2.2.2 - DLI

Flexible Engine Data Lake Insight for Flink is a cutting-edge cloud-based solution offered by Flexible Engine that combines the power of a data lake with the advanced data processing capabilities of Apache Flink. Apache Flink is an open-source stream processing framework that enables real-time data processing, analytics, and machine learning at scale. By integrating Flink with the Flexible Engine Data Lake Insight, organizations can unlock the full potential of their data lake by leveraging Flink's robust processing capabilities.

A data lake provides a centralized repository for storing vast amounts of structured, semi-structured, and unstructured data. It acts as a foundation for data exploration and analysis, allowing organizations to derive valuable insights from their data assets. With Flexible Engine Data Lake Insight for Flink, you can supercharge your data lake by harnessing the power of Flink's stream processing capabilities, enabling real-time analysis and actionable insights.

Objective is to create Flink jobs which will read the OBS Bucket where the OBS access logs are stored at  **nhe-obs-eu-w0-logs** in directory **nhe-obs-eu-w0-demo-log/** and push each OBS access entry to DMS Kafka


#### 2.2.2.1. Create Dedicated General Purpose Queue
 

- Create a **dedicated general purpose queue** to run the **Flink Jobs** 

Please make sure to tick **"For general purpose"** and **Dedicate Resource Mode**
Name it **"nhe_demo_fdedicated_queue"**


![[Pasted image 20230619145058.png]]

#### 2.2.2.2. Create VPC Peering

DLI Flink Job will read the data stored in OBS and push the data to DMS Kafka, therefore DLI needs to have access to VPC and Subnet through creating a VPC Peering

Creating a VPC Peering is completed through **"Datasource Connections"** within DLI
and select the "Queue" just created **"nhe_demo_fdedicated_queue"** and select the VPC (**nhe-vpc-demos(192.168.0.0/16)** and Subnet (**subnet-7096-dms-demo**) where the DMS Kafka Queue has been created.

![[Pasted image 20230619145341.png]]

Wait until the connection is **Active**

![[Pasted image 20230619145906.png]]

#### 2.2.2.3. Create Flink Job

- Create a DLI jobs to read OBS Logging file and push to DLI for persistence reason by selecting **Job Management / Flink Jobs / Create Job** and name it **nhe_obs2dms_json_v3**

![[Pasted image 20230619150716.png]]

- Copy / Paste the following code and make sure the 

``` FlinkSQL

CREATE SOURCE STREAM obs_source (
  ObsAccessEntry STRING
) WITH (
  type = "obs",
  region = "eu-west-0",
  bucket = "nhe-obs-eu-w0-logs",
  -- object_name = "nhe-obs-eu-w0-demo-log/2023-05-24-22-47-20-MTANPSRQGWNDDL5W",
  object_name = "nhe-obs-eu-w0-demo-log/",
  
  row_delimiter = "\n",
  field_delimiter = "\t"
);
CREATE SINK STREAM kafka_sink (
  ObsAccessEntry STRING
) -- output columns
WITH (
  type = "kafka",
  kafka_bootstrap_servers = "192.168.4.52:9092,192.168.4.45:9092,192.168.4.189:9092",
  -- ip/port of kafka cluster, you can use datasource connection to ensure it is connectable
  kafka_topic = "obsaccesslogs",
  -- topic to write into
  encode = "json" -- encode format, support "json/csv"
);
INSERT INTO
  kafka_sink
SELECT
  ObsAccessEntry
FROM
  obs_source;
```

- Make sure to select and hit save
	- CU : **20**
	- Queue **"nhe_demo_fdedicated_queue"**

- Click the "Check Semantics" button to validate the SQL synthax

![[Pasted image 20230619151104.png]]

Hit **Start*

![[Pasted image 20230619151149.png]]

### 2.2.3 - CSS

Flexible Engine Cloud Search Service (CSS) is an advanced cloud-based search solution offered by Flexible Engine, designed to simplify the process of building, managing, and scaling search capabilities within your applications. CSS enables organizations to leverage the power of search technology to deliver fast and accurate search experiences for their users, improving discoverability and driving user engagement.

Search functionality plays a crucial role in modern applications, allowing users to find relevant information quickly and efficiently. However, building and maintaining a robust search infrastructure can be complex and resource-intensive. Flexible Engine CSS addresses these challenges by providing a fully managed and scalable search service that eliminates the need for infrastructure setup and maintenance, enabling organizations to focus on delivering exceptional search experiences.

With Flexible Engine CSS, you can index and search large volumes of structured and unstructured data with ease. The service supports various data types, including text, numeric, geospatial, and more, allowing you to create comprehensive search experiences tailored to your specific requirements. CSS employs advanced indexing and retrieval algorithms, ensuring fast and accurate search results even across vast datasets.

#### 2.2.3.1. Create CSS Cluster

- Create Cluster version **7.10.2** using the VPC  (**nhe-vpc-demos**) named **nhe-css-demo-01**
- Make sure to select
	- HTTPS Access : **Enabled**
	- Public IP Address : **Automatically assign**
	- Administrator Username : **admin**
	- Administrator Password : **xxxxxxxxxx**

![[Pasted image 20230619152427.png]]
![[Pasted image 20230619152449.png]]

#### 2.2.3.2. Check the status

![[Pasted image 20230619152724.png]]

#### 2.2.3.3. Access Kibana

- Go to Cluster, **Kibana Public Access** 
![[Pasted image 20230619152820.png]]

- Click on **Kibana Public IP Address**

![[Pasted image 20230619153315.png]]

- Login using login : **admin** and password : **xxxxxxxxxx**

![[Pasted image 20230619153315.png]]

#### 2.2.3.4. Collect CSS Cluster Private Network Address

![[Pasted image 20230619153206.png]]

This will be used by Logstash script

### 2.2.4 - Logstash
#### 2.2.4.1. Create Ubuntu ECS

#### 2.2.4.2. Install Logstash on ECS
- Install Logstash **7.10.2-1** on Ubuntu **ECS 22.04**

The compatibliity version between,  Logstash and Kibana is Key
**Do not install latest version but specific version compatible with CSS Kibana Version**


```
wget -qO - https://artifacts.elastic.co/GPG-KEY-elasticsearch | sudo apt-key add -
sudo apt-get install apt-transport-https
echo "deb https://artifacts.elastic.co/packages/7.x/apt stable main" | sudo tee -a /etc/apt/sources.list.d/elastic-7.x.list
sudo apt-get update && sudo apt-get install logstash 

sudo apt list --all-versions logstash

```

#### 2.2.4.3. Connect Logstash ECS to CSS Elasticsearch

- Download the CSS Security Certificate file
- Upload it to ECS directory **/etc/logstash/conf.d/etc/logstash/conf.d/**

![[Pasted image 20230619153640.png]]

- Check that Logstash ECS can connect CSS Elasticsearch. Please use one of the Private Network IP from CSS

```bash
curl -XGET "https://192.168.0.116:9200" -u admin --cacert /etc/logstash/conf.d/CloudSearchService.cer

```

#### 2.2.4.4. Create Logstash Pipeline Configuration Script

- Create configuration file in **/etc/logstash/conf.d/dms2css.conf_json** (CSV to json)

```bash

sudo vi /home/cloud/logstash/pipeline/dms2css.conf_json
```

```LOGSTASH

input {
  kafka{
    codec => json
    bootstrap_servers => "192.168.4.52:9092,192.168.4.45:9092,192.168.4.189:9092"
    topics => ["obsaccesslogs"]
  }
}


filter {
  grok {
    match => {
      "ObsAccessEntry" => "%{NOTSPACE:BucketOwner} %{NOTSPACE:BucketName} \[%{HTTPDATE:Time}\] %{IPV4:Remote-IP} %{NOTSPACE:Requester} %{NOTSPACE:RequestID} %{NOTSPACE:Operation} %{NOTSPACE:Key} \"%{WORD:Request-URI-HttpMethod} %{URIPATHPARAM:Request-URI-Url} %{WORD:Request-URI-HttpVersion01}\/%{NOTSPACE:Request-URI-HttpVersion02}\" %{NOTSPACE:HTTPStatus} %{NOTSPACE:ErrorCode} %{NOTSPACE:BytesSent} %{NOTSPACE:ObjectSize} %{NOTSPACE:TotalTime} %{NOTSPACE:Turn-AroundTime} %{QS:Referer} %{QS:User-Agent} %{NOTSPACE:VersionID} %{NOTSPACE:STSLogUrn} %{NOTSPACE:StorageClass} %{NOTSPACE:TargetStorageClass} %{NOTSPACE:Misc}"
    }
  }

  date {
    match => ["Time","dd/MMM/yyyy:HH:mm:ss Z" ]
    target => "TransactionTime"
    }

  mutate {
    gsub => ["Referer","\"",""]
    gsub => ["User-Agent","\"",""]
    gsub => ["Misc","\"",""]
    add_field => {
      "Request-URI" => "%{Request-URI-HttpMethod} %{Request-URI-Url} %{Request-URI-HttpVersion01}/%{Request-URI-HttpVersion02}"
    }
    remove_field => ["Request-URI-HttpMethod","Request-URI-Url", "Request-URI-HttpVersion01","Request-URI-HttpVersion02"]

    convert => {
      "TotalTime" => "integer"
      "Turn-AroundTime" => "integer"
      "HTTPStatus" => "integer"
    }
  }


}


output {
  stdout {
        codec => json_lines
    }

  file {
        codec => json_lines
        path => "/home/cloud/logstash/logs/logstash_%{+YYYY-MM-dd}.log"
    }

  elasticsearch {
      hosts => ["https://192.168.0.116:9200","https://192.168.0.158:9200","https://192.168.0.245:9200"]
      index => "kibana_data_obs_access"
      user => "admin"
      password => "xxxxxx"
      cacert => "/home/cloud/logstash/settings/CloudSearchService.cer"
      document_id => "%{Requester}_%{RequestID}_%{TransactionTime}"
  }
}

```

Reference to Grok Filter : 
- Grok https://github.com/logstash-plugins/logstash-patterns-core/blob/main/patterns/legacy/grok-patterns

- Go to logstash directory

```bash
cd /usr/share/logstash/
```

- Run logstash command

```bash

sudo bin/logstash --log.level=debug -f /home/cloud/logstash/pipeline/dms2css.conf_json

```

### 2.2.5 - Elasticsearch & Kibana
#### 2.2.5.1. Elasticsearch

- Create the Elastisearch Index

```elasticsearch

PUT /kibana_data_obs_access

```

- Get number of document (row) in Index

```Elasticsearch

GET /kibana_data_obs_access/_count

```

