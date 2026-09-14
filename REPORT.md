# Hands-on L4 - Report

**Name:** Tarang Sonkusare  
**Student ID:** 801352372  
**Email:** tsonkusa@charlotte.edu  

---

## What I ran

I used the following commands in order. I did not modify the provided Java code.

```bash
docker --version
java -version
mvn -version
docker compose up -d
docker compose ps
mvn clean package
docker cp target/WordCountUsingHadoop-0.0.1-SNAPSHOT.jar resourcemanager:/tmp/
docker cp shared-folder/input/data/input.txt resourcemanager:/tmp/
docker exec -it resourcemanager bash
hdfs dfs -rm -r -f /input/data
hdfs dfs -rm -r -f /output
hdfs dfs -mkdir -p /input/data
hdfs dfs -put /tmp/input.txt /input/data/input.txt
hdfs dfs -ls /input/data
hdfs fsck /input/data/input.txt -files -blocks -locations
hadoop jar /tmp/WordCountUsingHadoop-0.0.1-SNAPSHOT.jar com.example.controller.Controller /input/data/input.txt /output
hdfs dfs -cat /output/part-r-00000
hdfs dfs -get /output /tmp/output-l4-rerun
exit
mv shared-folder/output shared-folder/output-old
mkdir shared-folder/output
docker cp resourcemanager:/tmp/output-l4-rerun/. shared-folder/output/
cat shared-folder/output/part-r-00000
rm -rf shared-folder/output-old
docker compose down
```

The `hdfs dfs -put` command copied the 939-byte local input file into HDFS. The NameNode recorded the file metadata and mapped the file to one HDFS block. The cluster used a replication factor of two, so two DataNodes stored replicas of that block. The `fsck` output reported one healthy block, two live replicas, three available DataNodes, and no missing or corrupt blocks.

---

## Input and output

### My input dataset

```text
Cloud computing allows organizations to process large amounts of data using flexible computing resources. Cloud platforms make computing resources available when applications need additional capacity. Data engineers use cloud systems to store data process data and analyze data for useful business decisions.

Hadoop provides distributed storage and distributed processing for large datasets. HDFS stores data across multiple DataNodes while Hadoop coordinates computing tasks across the cluster. When data enters HDFS the file is divided into blocks and those blocks are distributed across available DataNodes.

MapReduce processes data through map shuffle and reduce phases. The mapper reads input records and produces intermediate key value pairs. The shuffle groups every value associated with the same key before the reducer starts. The reducer receives each grouped key calculates the total count and writes the final output to HDFS.
```

### The output the job produced

```text
data	7
and	6
the	6
computing	4
The	3
key	3
distributed	3
across	3
blocks	2
large	2
process	2
Cloud	2
for	2
shuffle	2
Hadoop	2
reducer	2
available	2
HDFS	2
value	2
datasets.	1
through	1
with	1
DataNodes	1
calculates	1
organizations	1
amounts	1
reduce	1
cloud	1
flexible	1
starts.	1
associated	1
engineers	1
final	1
reads	1
allows	1
provides	1
coordinates	1
before	1
while	1
same	1
capacity.	1
business	1
total	1
phases.	1
multiple	1
additional	1
make	1
groups	1
are	1
output	1
DataNodes.	1
map	1
pairs.	1
intermediate	1
each	1
store	1
every	1
file	1
resources.	1
platforms	1
useful	1
divided	1
records	1
stores	1
storage	1
systems	1
resources	1
need	1
enters	1
input	1
processing	1
writes	1
cluster.	1
into	1
receives	1
grouped	1
produces	1
those	1
use	1
Data	1
decisions.	1
tasks	1
When	1
HDFS.	1
count	1
mapper	1
processes	1
MapReduce	1
analyze	1
using	1
applications	1
when	1
```

The pasted output above matches `shared-folder/output/part-r-00000` from this run.

---

## What I observed

Docker Compose started eight containers: the NameNode, three DataNodes, the ResourceManager, two NodeManagers, and the HistoryServer. The HDFS check showed that the 939-byte input occupied one block with two live replicas across the three available DataNodes. Its status was `HEALTHY`, and the check completed in 53 milliseconds.

The ResourceManager tracking URL identified application `application_1789340216283_0001`. At `http://localhost:8088`, the application completed with a successful final state, one map task, one reduce task, and 100% progress for both phases. The cluster ran two NodeManagers to execute YARN work. The job ran outside uber mode. Its progress changed from `map 0% reduce 0%`, to `map 100% reduce 0%`, and finally to `map 100% reduce 100%`. From the running-job message at 22:59:37 to successful completion at 23:00:35, the observed wall-clock job time was about 58 seconds. Hadoop reported 10,271 ms of map-task time and 13,843 ms of reduce-task time.

The job launched one map task and one reduce task. It read five input records and emitted 130 mapper records. The combiner reduced these to 92 records. The shuffle transferred 1,252 bytes and produced 92 reduce input groups. The reducer wrote 92 output records and 878 HDFS bytes. Hadoop read 1,040 HDFS bytes, completed one rack-local map task, and reported zero failed shuffles.

The output is sorted from the highest count to the lowest count by the provided reducer. The word `data` appeared seven times, while `and` and `the` each appeared six times. The results also demonstrate case sensitivity: `The` and `the`, `Cloud` and `cloud`, and `Data` and `data` were counted as different keys. Punctuation is retained because tokenization is based on whitespace, so `HDFS` and `HDFS.` are also different keys. Words shorter than three characters, such as `to`, were filtered out.

---

## Understanding the MapReduce cycle

The mapper read each input line and emitted every token containing at least three characters as a key paired with the value `1`. This produced 130 intermediate records from five input records. The combiner performed local aggregation before network transfer, reducing the 130 mapper records to 92 records and decreasing the amount of data sent through the shuffle.

During shuffle and sort, Hadoop grouped every intermediate value belonging to the same word key. Although the reducer task can be launched while mapping is underway, its reduce function cannot process a key until the required mapper output has been collected and grouped. In this run the progress log therefore showed `map 100% reduce 0%` before the reduce phase reached 100%. The reducer then summed the grouped values for each key and wrote 92 final word-count records to HDFS.

HDFS supported this cycle by storing the input as a replicated block on the DataNodes rather than as an ordinary file inside only the ResourceManager container. The NameNode maintained the file-to-block metadata, while the DataNodes stored the two replicas. The rack-local map counter indicates that Hadoop scheduled the mapper where the block was locally available within the cluster topology.

---

## Problems and fixes

After leaving the ResourceManager container, I accidentally attempted to execute the Hadoop job from the macOS terminal and received the exact error:

```text
zsh: command not found: hadoop
```

This happened because the Hadoop command-line tools and cluster configuration were installed inside the Docker containers, not directly on my Mac. I fixed the problem by entering the ResourceManager container with:

```bash
docker exec -it resourcemanager bash
```

I then ran the HDFS and Hadoop commands inside that container. The job completed successfully and produced the committed output file.
