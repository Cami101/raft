# raft-java
A study and implementation of raft in Java.

Based on the [Raft paper](https://github.com/maemual/raft-zh_cn) and the open-source Raft implementation by the Raft author, [LogCabin](https://github.com/logcabin/logcabin).

# Supported Features
* Leader election
* Log replication
* Snapshotting
* Dynamic cluster membership changes

## Quick Start
To deploy a 3-instance Raft cluster on a local machine, run the following script:
```
cd raft-java-example && sh deploy.sh
```
This script will deploy three instances (`example1`, `example2`, `example3`) in the `raft-java-example/env` directory.
It will also create a `client` directory for testing read and write operations in the Raft cluster.

### Testing Write Operations
After deployment, test writing to the cluster using:
```
cd env/client  
./bin/run_client.sh "list://127.0.0.1:8051,127.0.0.1:8052,127.0.0.1:8053" hello world
```

### Testing Read Operations
Run the following command:
```
./bin/run_client.sh "list://127.0.0.1:8051,127.0.0.1:8052,127.0.0.1:8053" hello
```

# Usage Guide
The following explains how to use the `raft-java` library to build a distributed storage system.

## Dependency Configuration (Not yet published to Maven Central, manual installation required)
```
<dependency>
    <groupId>com.github.raftimpl.raft</groupId>
    <artifactId>raft-java-core</artifactId>
    <version>1.9.0</version>
</dependency>
```

## Define Data Write and Read Interfaces
```protobuf
message SetRequest {
    string key = 1;
    string value = 2;
}
message SetResponse {
    bool success = 1;
}
message GetRequest {
    string key = 1;
}
message GetResponse {
    string value = 1;
}
```
```java
public interface ExampleService {
    Example.SetResponse set(Example.SetRequest request);
    Example.GetResponse get(Example.GetRequest request);
}
```

## Server-Side Usage
### 1. Implement the StateMachine Interface
```java
// This interface contains three methods mainly called by Raft internally
public interface StateMachine {
    /**
     * Takes a snapshot of the data in the state machine, called periodically on each node
     * @param snapshotDir directory for snapshot output
     */
    void writeSnapshot(String snapshotDir);
    
    /**
     * Loads a snapshot into the state machine, called when a node starts
     * @param snapshotDir snapshot directory
     */
    void readSnapshot(String snapshotDir);
    
    /**
     * Applies data to the state machine
     * @param dataBytes binary data
     */
    void apply(byte[] dataBytes);
}
```

### 2. Implement Data Write and Read Interfaces
```java
// ExampleService implementation contains these members
private RaftNode raftNode;
private ExampleStateMachine stateMachine;
```

```java
// Data write logic
byte[] data = request.toByteArray();
// Synchronize data write into the Raft cluster
boolean success = raftNode.replicate(data, Raft.EntryType.ENTRY_TYPE_DATA);
Example.SetResponse response = Example.SetResponse.newBuilder().setSuccess(success).build();
```

```java
// Data read logic, implemented by the specific application state machine
Example.GetResponse response = stateMachine.get(request);
```

### 3. Server Startup Logic
```java
// Initialize RPCServer
RPCServer server = new RPCServer(localServer.getEndPoint().getPort());
// Application state machine
ExampleStateMachine stateMachine = new ExampleStateMachine();
// Set Raft options, for example:
RaftOptions.snapshotMinLogSize = 10 * 1024;
RaftOptions.snapshotPeriodSeconds = 30;
RaftOptions.maxSegmentFileSize = 1024 * 1024;
// Initialize RaftNode
RaftNode raftNode = new RaftNode(serverList, localServer, stateMachine);
// Register Raft consensus service for inter-node communication
RaftConsensusService raftConsensusService = new RaftConsensusServiceImpl(raftNode);
server.registerService(raftConsensusService);
// Register Raft client service
RaftClientService raftClientService = new RaftClientServiceImpl(raftNode);
server.registerService(raftClientService);
// Register application-specific service
ExampleService exampleService = new ExampleServiceImpl(raftNode, stateMachine);
server.registerService(exampleService);
// Start RPCServer and initialize Raft node
server.start();
raftNode.init();
```

