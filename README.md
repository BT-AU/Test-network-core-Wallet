# Test-network-core-DAPP
For the core DAPP of the test network, all generated tokens cannot continue to be traded
It is a blockchain performance testing framework, which can be used to test different blockchain implementations. support
fabric v1.0+
sawtooth 1.0+
Iroha 1.0
Test contents and indicators include:
Transaction success rate
Transaction throughput TPS
Transaction delay
resource consumption 
Feel it first
get ready
First install nodejs 8. X, node gyp, docker and docker compose.
git clone  https://github.com/hyperledger/caliper.git
cd caliper
npm install
one
two
three
Install the blockchain SDK (take fabric as an example)
#Caliper project directory
npm install  grpc@1.10.1  fabric-ca-client fabric-client
one
two
Run a test
The performance test examples are in the benchmark directory. The usage is as follows:
node benchmark/simple/main.js -c yourconfig.json -n yournetwork.json
one
-C is used to specify the configuration file of the blockchain. If not specified, it defaults to config.json;
-N is used to specify the blockchain network configuration file. If not specified, it is defined by the configuration file specified by - C.
An example of running a smallbank:
node benchmark/smallbank/main.js
one
The generated report is as follows (partial):
Blockchain performance testing tool caliper_ hyperledger
framework
The core of this standard framework is the "adaptation layer" that can interpret information, so that caliper can install smart contracts, trigger contracts, or query the status of various distributed ledgers, so as to better measure their effectiveness.
Blockchain performance testing tool caliper_ Blockchain_ 02
Adaptation layer
The adaptation layer is used to integrate the existing blockchain system with the caliper framework. The adapter uses the corresponding chain SDK and API to implement caliper blockchain NBIs.
Interface &amp; core layer
Blockchain operation interface: including deployment contract, call contract, query ledger status and other operations;
Resource monitoring: monitor docker container and local processes, including CPU, memory, network IO, etc;
Performance analysis: read predefined performance data (such as TPS, delay, success rate, etc.) and print the report. These data will be recorded when calling NBI (such as creation time, transaction submission time, transaction result, etc.);
Generate HTML reports.
application layer
The application layer runs the blockchain test scenario. Each test scenario is defined by a configuration file, including the configuration and test parameters of the underlying blockchain network.
A default blockchain engine is built in the project. Of course, developers can also define their own blockchain engine based on NBI.
Blockchain engine
Blockchain performance testing tool caliper_ Blockchain_ 03
configuration file
Take benchmark / simple / config.json as an example:
{
  "blockchain": {
    "type": "fabric",
    "config": "benchmark/simple/fabric.json"
  },
  "command" : {
    "start": "docker-compose -f network/fabric/simplenetwork/docker-compose.yaml up -d",
    "end" : "docker-compose -f network/fabric/simplenetwork/docker-compose.yaml down;docker rm $(docker ps -aq);docker rmi $(docker images dev* -q)"
  },
  "test": {
    "name": "simple",
    "description" : "This is an example benchmark for caliper, to test the backend DLT's performance with simple account opening & querying transactions",
    "clients": {
      "type": "local",
      "number": 5
    },
    "rounds": [{
        "label" : "open",
        "txNumber" : [1000, 1000, 1000],
        "rateControl" : [{"type": "fixed-rate", "opts": {"tps" : 50}}, {"type": "fixed-rate", "opts": {"tps" : 100}}, {"type": "fixed-rate", "opts": {"tps" : 150}}],
        "arguments": { "money": 10000 },
        "callback" : "benchmark/simple/open.js"
      },
      {
        "label" : "query",
        "txNumber" : [5000, 5000],
        "rateControl" : [{"type": "fixed-rate", "opts": {"tps" : 100}}, {"type": "fixed-rate", "opts": {"tps" : 200}}],
        "callback" : "benchmark/simple/query.js"
      }]
  },
  "monitor": {
    "type": ["docker", "process"],
    "docker":{
      "name": ["all"]
    },
    "process": [
      {
        "command" : "node",
        "arguments" : "local-client.js",
        "multiOutput" : "avg"
      }
    ],
    "interval": 1
  }
}Blockchain defines the type of blockchain to be tested and gives specific configuration files;
Command defines the commands at the beginning and end of the test;
Test defines test related information;
Monitor defines how to monitor resource objects.
Master
The test flow implemented by the master consists of three stages:
Preparation stage: create and initialize blockchain, deploy smart contract and start monitoring;
Test phase: start a loop test, the test task will be assigned to the client for execution, and the client will return performance test data;
Report stage: analyze test data and generate reports in HTML format.
Order
Client
Local client
Because node.js is inherently single threaded, it will fork multiple local client sub processes to make full use of multi-core and improve test efficiency. Each sub process runs a blockchain client.
Zookeeper client
Multiple zookeeper clients are started independently. After starting, they will register themselves and stand by for test tasks. After testing, a znode containing result data will be created. It also forks multiple child processes (local clients).
User defined test module
Functions for generating and submitting transactions are defined (return values are promise):
Init: it will be called by the client before each round of test;
Run: defines how transactions are executed. The client will call it circularly according to the workload definition;
End: it is called after each round of test, and some cleaning work is usually defined.
Roll up the source code
Still based on the above architecture diagram, this time from top to bottom.
The first is the benchmark layer
Starting with the test command, take smallbank as an example:
node benchmark/smallbank/main.js
one
The test cases are located in the benchmark / directory. It is written by the tester and configured in test. Rounds [. Callback] of the configuration file specified by - C.
In main.js, you load two configuration files, and then call the run method of src/comm/bench-flow.js to transfer the two configuration files into:
const framework = require('../../src/comm/bench-flow.js');
framework.run(absConfigFile, absNetworkFile);
one
two
Bench-flow.js can be regarded as a test engine, and the run method defines a complete test process.
Below is the interface and core layer
The interface, blockchain NBIs, is located in Src / comm / blockchain-interface.js. The blockchain interface defines the basic operations of the blockchain:
Blockchain performance testing tool caliper_ Blockchain_ 04
In order to generate test reports, monitoring must be provided to collect data, mainly focusing on the monitoring of docker and process. The code files include monitor-interface.js, monitor-docker.js and monitor-process.js. From the name, we can see the interface and two specific implementations. Let's see what the interface defines:
Blockchain performance testing tool caliper_ Blockchain_ 05
In addition to start and stop operations, it mainly focuses on the monitoring of CPU, memory and network.
The data statistics and report generation functions are located in Src / comm / report.js.
Blockchain performance testing tool caliper_ Blockchain_ 06
Adaptation layer
Caliper supports the testing of fabric, sawtooth, etc. according to the architecture diagram, it also wants to support Ethereum, but it has not yet.
The specific adaptation codes are located in Src / fabric, Src / sawtooth, Src / iroha and other directories. For example, the fabric class in fabric.js inherits from blockchaininterface to realize the functions of initialization, installation / execution of smart contracts and so on.
Blockchain performance testing tool caliper_ Blockchain_ 07
These functions naturally call fabric client internally, so you need to install them before testing:
