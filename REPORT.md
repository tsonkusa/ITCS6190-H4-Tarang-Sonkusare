# Hands-on L4 — Report

**Name:** Tarang Sonkusare  
**Student ID:** 801352372  
**Email:** tsonkusa@charlotte.edu  

---

## What I ran

I used the following commands in the order provided by the assignment instructions:

```bash
docker --version
java -version
mvn -version
docker compose up -d
mvn clean package
docker cp target/WordCountUsingHadoop-0.0.1-SNAPSHOT.jar resourcemanager:/tmp/
docker cp shared-folder/input/data/input.txt resourcemanager:/tmp/
docker exec -it resourcemanager bash
cd /tmp
hadoop fs -mkdir -p /input/data
hadoop fs -put ./input.txt /input/data
hadoop fs -ls /input/data
hadoop jar /tmp/WordCountUsingHadoop-0.0.1-SNAPSHOT.jar com.example.controller.Controller /input/data/input.txt /output
hadoop fs -cat /output/*
hdfs dfs -get /output /tmp/
exit
docker cp resourcemanager:/tmp/output/. shared-folder/output/
docker compose down
```

I followed the steps in the README and did not change the provided Java code.

---

## Input and output

### My input dataset

```text
Cloud computing processes data
Cloud computing stores data
Hadoop processes big data
Hadoop processes cloud data
MapReduce counts words
MapReduce processes data
Data processing is useful
```

### The output the job produced

```text
data	5
processes	4
Cloud	2
Hadoop	2
MapReduce	2
computing	2
Data	1
useful	1
stores	1
counts	1
cloud	1
processing	1
words	1
big	1
```

---

## What I observed

The Hadoop cluster started successfully, and the NameNode page showed three live DataNodes. When I ran the job, the map phase reached 100% before the reduce phase reached 100%. Both phases took around the same amount of time, and the job completed successfully.

The output was arranged from the most frequently used words to the least frequently used words. The word `data` appeared five times, while `processes` appeared four times. I also noticed that the program is case-sensitive because `Cloud` and `cloud` were counted separately. The same happened with `Data` and `data`. The word `is` was not included because it has fewer than three characters.

---

## Problems and fixes

One problem I experienced was that an older container version caused issues when I tried to run the Hadoop cluster. I resolved the problem by updating Docker and using the current container version provided in the repository. After updating it, all of the Hadoop containers started correctly, and the MapReduce job completed successfully.