# kafka-alarm-system
An alarm system built on [Kafka](https://kafka.apache.org/) that supports pluggable alarm sources.  This project ties together all of the message pipeline services that make up the alarm system in a docker-compose file and adds an alarm system console Docker image containing Python scripts for configuring and interacting with the system.

---
- [Overview](https://github.com/JeffersonLab/kafka-alarm-system#overview)
- [Quick Start with Docker](https://github.com/JeffersonLab/kafka-alarm-system#quick-start-with-docker)
- [Alarm System Console](https://github.com/JeffersonLab/kafka-alarm-system#alarm-system-console)
---

## Overview
The alarm system console, included in this project, provides scripts to manage the Kafka topics and their schemas needed for the alarm system. 

Alarms are triggered by producing messages on the __active-alarms__ topic, which is generally done programmatically via Kafka Connect or Kafka Streams services.  For example EPICS alarms could be handled by the additional service: [epics2kafka](https://github.com/JeffersonLab/epics2kafka).  Anything can produce messages on the active-alarms topic (with proper authorization).

An [Operator Graphical User Interface to the Alarm System](https://github.com/JeffersonLab/graphical-alarm-client) provides a convenient desktop app for operators to view active alarms, see alarm definitions (registered-alarms), and shelve active alarms.

TODO: An admin web interface provides a convenient app for admins to manage the list of all possible alarms and their definitions.

A Kafka Streams app [acknowledgements2epics](https://github.com/JeffersonLab/acknowledgements2epics) performs an EPICS alarm acknowledgement on alarms of provider __DirectCAAlarm__, necessary for alarms which have latched.

TODO: A Kafka Streams app to manage the shelved-alarms topic (The Shelf Service).   The shelf service looks for expired shelved messages and unsets them with tombstone records as a matter of book-keeping to speed up processing for clients (on a compacted topic tombstone records keep most recent message list short).  The shelf service will also monitor active-alarms topic and ensure that channels can only be shelved if they are active - automatically cleanup (tombstone) shelved messages for channels that are no longer in active alarm. 

## Quick Start with Docker 
1. Grab project
```
git clone https://github.com/JeffersonLab/kafka-alarm-system
cd kafka-alarm-system
```
2. Launch Docker
```
docker-compose up
```
3. Monitor active alarms
```
docker exec -it console /scripts/active-alarms/list-active.py --monitor
```
4. Trip an alarm  
```
docker exec console /scripts/active-alarms/set-active.py channel1 --priority P1_LIFE
```
[Scripts Reference](https://github.com/JeffersonLab/kafka-alarm-system/wiki/Scripts-Reference)

The alarm system is composed of the following services:
   - Kafka - distributed message system
   - [ZooKeeper](https://zookeeper.apache.org/) - required by Kafka for bookkeeping and coordination
   - [Schema Registry](https://github.com/confluentinc/schema-registry) - message schema lookup
   - Alarm Console - defined in this project; provides Python scripts to setup and interact with the alarm system

**Note**: The docker-compose services require significant system resources - tested with 4 CPUs and 4GB memory.

## Alarm System Console

### Kafka Topics
The alarm system state is stored in three Kafka topics.   Topic schemas are stored in the [Schema Registry](https://github.com/confluentinc/schema-registry) in [AVRO](https://avro.apache.org/) format.  Python scripts are provided for managing the alarm system topics.  

| Topic | Description | Key Schema | Value Schema | Scripts |
|-------|-------------|------------|--------------|---------|
| registered-alarms | Set of all possible alarm metadata (descriptions). | String: alarm name | AVRO: [registered-alarms-value](https://github.com/JeffersonLab/kafka-alarm-system/blob/master/schemas/registered-alarms-value.avsc) | set-registered.py, list-registered.py |
| active-alarms | Set of alarms currently active (alarming). | String: alarm name | AVRO: [active-alarms-value](https://github.com/JeffersonLab/kafka-alarm-system/blob/master/schemas/active-alarms-value.avsc) | set-active.py, list-active.py |
| shelved-alarms | Set of alarms that have been shelved. | String: alarm name | AVRO: [shelved-alarms-value](https://github.com/JeffersonLab/kafka-alarm-system/blob/master/schemas/shelved-alarms-value.avsc) | set-shelved.py, list-shelved.py |

The alarm system relies on Kafka not only for notification of changes, but for [Event Sourcing](https://martinfowler.com/eaaDev/EventSourcing.html) - everything is stored in Kafka and the entire state of
the system is built by replaying messages.   All topics have compaction enabled to remove old messages that would be overwritten on replay.  Compaction is not very aggressive though so some candidates for deletion are often lingering when clients connect so they must be prepared to handle the ordered messages on replay as ones later in the stream may overwrite ones earlier.

To unset (remove) a record use the --unset option with the "set" scripts, This writes a null/None tombstone record.  To modify a record simply set a new one with the same key as the message stream is ordered and newer records overwrite older ones.  To see all options use the --help option. 

### Message Metadata
The alarm system topics are expected to include audit information in Kafka message headers:

| Header | Description |
|--------|-------------|
| user | The username of the account whom produced the message |
| producer | The application name that produced the message |
| host | The hostname where the message was produced |

Additionally, the built-in timestamp provided in all Kafka messages is used to provide basic message timing information.  The alarm system uses the default broker provided timestamp (as opposed to producer provided).

**Note**: There is no schema for message headers so content is not easily enforceable.  However, the topic management scripts provided include the audit headers listed.

### Docker
A docker image containing scripts can be built from the Dockerfile included in the project.  To build within a network using man-in-the-middle network scanning (self-signed certificate injection) you can provide an optional build argument pointing to the custom CA certificate file (pip will fail to download dependencies if certificates can't be verified).   For example:
```
docker build -t console . --build-arg CUSTOM_CRT_URL=http://pki.jlab.org/JLabCA.crt
```
Grab Image from [DockerHub](https://hub.docker.com/r/slominskir/kafka-alarm-system):
```
docker pull slominskir/kafka-alarm-system
```

### Python Environment
Scripts tested with Python 3.7

Generally recommended to use a Python virtual environment to avoid dependency conflicts.  You can use the requirements.txt file to ensure the Python module dependences are installed:

```bash
python3 -m venv env
source env/bin/activate
pip install -r requirements.txt
```

*Note*: [Jefferson Lab Internal Proxy](https://github.com/JeffersonLab/kafka-alarm-scripts/wiki/JeffersonLabProxy)

### Configure
By default the scripts assume you are executing them on the same box as a standalone Kafka with and Schema Registry.  To modify the defaults set the following environment variables:

| Variable | Default |
|----------|---------|
| BOOTSTRAP_SERVERS | `localhost:9092` |
| SCHEMA_REGISTRY | `http://localhost:8081` |
