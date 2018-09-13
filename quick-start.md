---
layout: doc
title:  "Quick Start" 
permalink: /docs/quick-start.html
---

Guide To Install **Apache Griffin 0.3.0-incubating** On Hortonworks sandbox. 

To take a quick start of Apache Griffin, we can try it [in docker](#in-docker), or [at local](#at-local) if spark environment is prepared.

## In Docker

### Environment Preparation
You can use our pre-built docker images as the environment.
1.  Install [docker](https://docs.docker.com/engine/installation/) and [docker compose](https://docs.docker.com/compose/install/).
2.  Increase vm.max_map_count of your local machine(linux), to use elasticsearch.
    ```
    sysctl -w vm.max_map_count=262144
    ```
    - For macOS, please increase enough memory available for docker (For example, set more than 4 GB in docker->preferences->Advanced) or decrease memory for es instance(For example, set -Xms512m -Xmx512m in jvm.options)  - For other platforms, please reference to this link from elastic.co
    [max_map_count kernel setting](https://www.elastic.co/guide/en/elasticsearch/reference/current/docker.html)
3.  Pull griffin pre-built docker images, but if you access docker repository easily(NOT in China).
    ```
    docker pull apachegriffin/griffin_spark2:0.3.0
    docker pull apachegriffin/elasticsearch
    ```
    For Chinese users, you can pull the images from the following mirrors.
    ```
    docker pull registry.docker-cn.com/apachegriffin/griffin_spark2:0.3.0
    docker pull registry.docker-cn.com/apachegriffin/elasticsearch
    ```
    The docker images are the griffin environment images.
    - `apachegriffin/griffin_spark2`: This image contains mysql, hadoop, hive, spark, livy, griffin service, griffin measure, and some prepared demo data, it works as a single node spark cluster, providing spark engine and griffin service.
    - `apachegriffin/elasticsearch`: This image is based on official elasticsearch, adding some configurations to enable cors requests, to provide elasticsearch service for metrics persist.

### Run Docker Image
Run the docker images.
1.  Copy [docker-compose-batch.yml](https://github.com/apache/incubator-griffin/blob/master/griffin-doc/docker/compose/docker-compose-batch.yml) to your work path.
2.  In your work path, start docker containers by using docker compose, wait for about one minutes, then griffin service is ready.
    ```
    docker-compose -f docker-compose-batch.yml up -d
    ```
3.  Login to the griffin container.
    ```
    docker exec -it <griffin docker container id> bash
    cd ~/measure
    ```

### Data Preparation
Prepare the test data in Hive.
In the griffin docker image, we've prepared two Hive tables named `demo_src` and `demo_tgt`, and the test data is generated hourly.
The schema is like this:
```
id                      bigint                                      
age                     int                                         
desc                    string                                      
dt                      string                                      
hour                    string 
```
In which `dt` and `hour` are the partition columns, with string values like `20180912` and `06`.

### Configuration Files
Prepare the configuration files, env.json for the common configuration, and dq.json for the definition of measurement.
**env.json**
```
{
  "spark": {
    "log.level": "WARN"
  },
  "sinks": [
    {
      "type": "console"
    },
    {
      "type": "hdfs",
      "config": {
        "path": "hdfs:///griffin/persist"
      }
    },
    {
      "type": "elasticsearch",
      "config": {
        "method": "post",
        "api": "http://es:9200/griffin/accuracy"
      }
    }
  ]
}
```
**dq.json**
```
{
  "name": "batch_accu",
  "process.type": "batch",
  "data.sources": [
    {
      "name": "src",
      "baseline": true,
      "connectors": [
        {
          "type": "hive",
          "version": "1.2",
          "config": {
            "database": "default",
            "table.name": "demo_src"
          }
        }
      ]
    }, {
      "name": "tgt",
      "connectors": [
        {

          "type": "hive",
          "version": "1.2",
          "config": {
            "database": "default",
            "table.name": "demo_tgt"
          }
        }
      ]
    }
  ],
  "evaluate.rule": {
    "rules": [
      {
        "dsl.type": "griffin-dsl",
        "dq.type": "accuracy",
        "out.dataframe.name": "accu",
        "rule": "src.id = tgt.id AND src.age = tgt.age AND src.desc = tgt.desc",
        "details": {
          "source": "src",
          "target": "tgt",
          "miss": "miss_count",
          "total": "total_count",
          "matched": "matched_count"
        },
        "out": [
          {
            "type": "metric",
            "name": "accu"
          },
          {
            "type": "record",
            "name": "missRecords"
          }
        ]
      }
    ]
  },
  "sinks": ["CONSOLE", "HDFS"]
}
```
In griffin docker container, the two configuration files are also prepared as `~/json/env.json` and `~/json/batch-accu-config.json`.

### Submit Measure Job
Submit the measure job to Spark, with the configuration files as parameters.
```
spark-submit --class org.apache.griffin.measure.Application --master yarn --deploy-mode client --queue default \
--driver-memory 1g --executor-memory 1g --num-executors 2 \
~/measure/griffin-measure.jar \
~/json/env.json ~/json/batch-accu-config.json
```
Then you can get the application log in console, after the job finishes, the metrics will be printed like this: 
```
batch_accu [1536802876892] metrics: 
{"name":"batch_accu","tmst":1536802876892,"value":{"total_count":375000,"miss_count":1484,"matched_count":373516}}
```
The result will also be saved in HDFS: `hdfs:///griffin/persist/batch_accu/1536802876892/`, with metrics in `_METRICS` file, and the mis-matched items in `missRecords` file.

## At Local

### Environment Preparation
You need to prepare the environment for Apache Griffin measure module, including the following softwares:
- JDK (1.8+)
- Hadoop (2.6.0+)
- Spark (2.2.1+)
- Hive (2.2.0)

### Build Griffin Measure Module
1.  Download Griffin source package [here](https://www.apache.org/dist/incubator/griffin/0.3.0-incubating).
2.  Unzip the source package.
    ```
    unzip griffin-0.3.0-incubating-source-release.zip
    cd griffin-0.3.0-incubating-source-release
    ```
3.  Build Griffin jars.
    ```
    mvn clean install
    ```
    Move the built griffin measure jar to your work path.
    ```
    mv measure/target/measure-0.3.0-incubating.jar <work path>/griffin-measure.jar
    ```

### Data Preparation
Prepare your test data in Hive, you can refer to the [test data prepared in docker](#data-preparation).

### Configuration Files
Prepare the configuration files, you can refer to the [configuration files in docker](#configuration-files).

### Submit Measure Job
Submit the measure job to Spark, with the configuration files as parameters.
```
spark-submit --class org.apache.griffin.measure.Application --master yarn --deploy-mode client --queue default \
--driver-memory 1g --executor-memory 1g --num-executors 2 \
<path>/griffin-measure.jar \
<path>/env.json <path>/batch-accu-config.json
```
Then you can get the application log in console, after the job finishes, the metrics will be printed like this: 
```
batch_accu [1536802876892] metrics: 
{"name":"batch_accu","tmst":1536802876892,"value":{"total_count":375000,"miss_count":1484,"matched_count":373516}}
```
The result will also be saved in HDFS: `hdfs:///griffin/persist/batch_accu/1536802876892/`, with metrics in `_METRICS` file, and the mis-matched items in `missRecords` file.

## More Details
For more details about griffin measures, you can visit our documents in [github](https://github.com/apache/incubator-griffin/tree/master/griffin-doc).