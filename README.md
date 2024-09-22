# raft-distributed-system

# raft-java
Raft implementation library for Java.<br>
Referenced from [Raft Paper](https://github.com/maemual/raft-zh_cn) and the open-source implementation [LogCabin](https://github.com/logcabin/logcabin) by Raft's author.

# Supported Features
* Leader election
* Log replication
* Snapshot
* Dynamic cluster membership changes

## Quick Start
To deploy a Raft cluster with 3 instances locally, execute the following script:<br>
`cd raft-java-example && sh deploy.sh`<br>
This script will deploy three instances (example1, example2, example3) under the `raft-java-example/env` directory.<br>
It will also create a `client` directory for testing the read and write capabilities of the Raft cluster.<br>
Once successfully deployed, test the write operation with the following script:<br>
`cd env/client`<br>
`./bin/run_client.sh "list://127.0.0.1:8051,127.0.0.1:8052,127.0.0.1:8053" hello world`<br>
Test the read operation with the following command:<br>
`./bin/run_client.sh "list://127.0.0.1:8051,127.0.0.1:8052,127.0.0.1:8053" hello`

# Usage
Below is an introduction to how to use the `raft-java` dependency library to implement a distributed storage system.

## Add Dependency (Not yet published to Maven Central Repository, manual installation required)
```xml
<dependency>
    <groupId>com.github.raftimpl.raft</groupId>
    <artifactId>raft-java-core</artifactId>
    <version>1.9.0</version>
</dependency>
```

## Define Data Write and Read Interface
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

## Server-side Usage
1. Implement the StateMachine interface.
```java
// This interface has three main methods for Raft's internal calls
public interface StateMachine {
    /**
     * Perform a snapshot of data in the state machine, called periodically by each node locally.
     * @param snapshotDir Directory to output the snapshot data.
     */
    void writeSnapshot(String snapshotDir);
    
    /**
     * Load the snapshot into the state machine, called when the node starts up.
     * @param snapshotDir Directory containing the snapshot data.
     */
    void readSnapshot(String snapshotDir);
    
    /**
     * Apply data to the state machine.
     * @param dataBytes Data in binary format.
     */
    void apply(byte[] dataBytes);
}
```

2. Implement Data Write and Read Interfaces
```
// In the ExampleService implementation class, include the following members:
private RaftNode raftNode;
private ExampleStateMachine stateMachine;
```
```
// Main logic for data write:
byte[] data = request.toByteArray();
// Write data synchronously to the Raft cluster:
boolean success = raftNode.replicate(data, Raft.EntryType.ENTRY_TYPE_DATA);
Example.SetResponse response = Example.SetResponse.newBuilder().setSuccess(success).build();
```
```
// Main logic for data read, implemented by the specific application state machine:
Example.GetResponse response = stateMachine.get(request);
```

3. Server Startup Logic
```
// Initialize the RPCServer
RPCServer server = new RPCServer(localServer.getEndPoint().getPort());
// Application state machine
ExampleStateMachine stateMachine = new ExampleStateMachine();
// Set Raft options, for example:
RaftOptions.snapshotMinLogSize = 10 * 1024;
RaftOptions.snapshotPeriodSeconds = 30;
RaftOptions.maxSegmentFileSize = 1024 * 1024;
// Initialize RaftNode
RaftNode raftNode = new RaftNode(serverList, localServer, stateMachine);
// Register services for Raft nodes to call each other
RaftConsensusService raftConsensusService = new RaftConsensusServiceImpl(raftNode);
server.registerService(raftConsensusService);
// Register Raft services for client calls
RaftClientService raftClientService = new RaftClientServiceImpl(raftNode);
server.registerService(raftClientService);
// Register services provided by the application itself
ExampleService exampleService = new ExampleServiceImpl(raftNode, stateMachine);
server.registerService(exampleService);
// Start the RPCServer and initialize the Raft node
server.start();
raftNode.init();
```
