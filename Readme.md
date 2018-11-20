## Test suite for Kubernetes

This test suite consists of two Helm charts for network bandwith testing and load testing a Kuberntes cluster.
The structure of the included charts looks a follows:
````
k8s-testsuite/
- load-test/
|____Chart.yaml
|____values.yaml
|____.helmignore
|____templates/
- network-test/
|____Chart.yaml
|____values.yaml
|____.helmignore
|____templates/
```` 

You can install a test by running:
```` 
> git clone https://github.com/mrahbar/k8s-testsuite.git
> cd k8s-testsuite
> helm install --namespace load-test ./load-test
> helm install --namespace network-test ./network-test
```` 


----------


### Load test 

This Helm chart deployes a full load test suite in Kubernetes. It consists of the 3 microservices:
1. A webserver based on [simple-webserver](https://github.com/mrahbar/simple-webserver)
2. A loadbot client which is based on [this](https://github.com/kubernetes/contrib/tree/master/scale-demo/vegeta) and [that](https://github.com/tsenart/vegeta)
3. An [aggregator](https://github.com/mrahbar/kubespector/tree/master/resources/scaletest) which orchestrates the test run

##### Aggregator

Since the webservers and the loadbots work autonomic the task of the aggregator ist to orchestrate the test run. 
It does this by useing the Kubernetes api via [client-go](https://github.com/kubernetes/client-go) library to talk set up the desires replicas of each unit.
The test run consists of the following tests scenarios:

| Szenario | Loadbots | Webserver |
| ------------- |:-------------:| -----:|
|Idle|1|1|
|Under load | 1|10|
|Equal load| 10|10|
|Over load|100|10|
|High load|100|100|

The maximum count of replicas (default 100) can be set with `--set aggregator.maxReplicas=...`.   

##### Loadbots
The loadbots have the task to run a predefined level of queries per second. Vegeta publishes [detailed statistics](https://github.com/tsenart/vegeta#json) which will be fetched and evaluated by the aggregator. This metrics are:
 - Queries-Per-Second (QPS)
 - Success-Rate
 - Mean latency
 - 99th percentile latency

#### Test results
When all tests are finishes the aggregator will print the following summary to its logs:

> GENERATING SUMMARY OUTPUT
> Summary of load scenarios:
> 0. Idle      : QPS: 10037    Success: 100.00  % Latency: 949.82µs (mean) 3.004154ms (99th)
> 1. Under load: QPS: 10014    Success: 100.00  % Latency: 965.549µs (mean) 1.985838ms (99th)
> 2. Equal load: QPS: 50078    Success: 100.00  % Latency: 982.519µs (mean) 7.213018ms (99th)
> 3. Over load : QPS: 501302   Success: 100.00  % Latency: 198.21451ms (mean) 859.504601ms (99th)
> 4. High load : QPS: 502471   Success: 100.00  % Latency: 239.26364ms (mean) 1.018523444s (99th) 
> END SUMMARY DATA

##### Configuration

The following table lists the configurable parameters of the chart and their default values.

Parameter | Description | Default
--------- | ----------- | -------
`cpuRequests.webserver` | Memory request of each webserver pod | 100m
`cpuRequests.loadbot` | Memory request of each loadbots  pod | 100m
`aggregator.maxReplicas` | Maximum replicas for ReplicationController| 100
`loadbot.rate` | QPS of each loadbot. [Docs](https://github.com/tsenart/vegeta#-rate) | 1000
`loadbot.workers` | Initial number of workers used in the attack. [Docs](https://github.com/tsenart/vegeta#-workers) | 10
`loadbot.duration` | Duration of each attack. [Docs](https://github.com/tsenart/vegeta#-duration) | 1s
`images.*Version` | Image version for *loadbot*, *webserver* and *aggregator* | 1.0
`imagePullPolicy` | Whether to Always pull imaged or only IfNotPresent | IfNotPresent
`rbac.create` | Create rbac rules for aggregator | true
`rbac.serviceAccountName` | rbac.create should be false to use this serviceAccount | 

----------


### Network test 

This Helm chart deployes a network test suite in Kubernetes. It consists of the 2 microservices:
1. An orchestrator
2. A worker launched three times

Both services are run from the same image either as `--mode=orchestrator` or `--mode=worker`. The services are bases on [k8s-nptest](https://github.com/mrahbar/k8s-nptest) and use iperf3 and netperf-2.7.0 internally. 

##### Orchestrator
The orchestrator pod coordinates the worker pods to run tests in serial order for the 4 scenarios described below. Using pod affinity rules Worker Pods 1 and 2 are placed on the same Kubernetes node, and Worker Pod 3 is placed on a different node. The nodes all communicate with the orchestrator pod service using simple golang rpcs and request work  items. **A minimum of two Kubernetes worker nodes are necessary for this test.**

##### Test scenario 
Five major network traffic paths are combination of Pod IP vs Virtual IP and whether the pods are co-located on the same node versus a remotely located pod.

1. Same VM using Pod IP: Same VM Pod to Pod traffic tests from Worker 1 to Worker 2 using its Pod IP.

2. Same VM using Cluster/Virtual IP: Same VM Pod to Pod traffic tests from Worker 1 to Worker 2 using its Service IP (also known as its Cluster IP or Virtual IP).

3. Remote VM using Pod IP: Worker 3 to Worker 2 traffic tests using Worker 2 Pod IP.

4. Remote VM using Cluster/Virtual IP: Worker 3 to Worker 2 traffic tests using Worker 2 Cluster/Virtual IP.

5. Same VM Pod Hairpin: Worker 2 to itself using Cluster IP

For each test the MTU (MSS tuning for TCP and direct packet size tuning for UDP) will be linearly increased from 96 to 1460 in steps of 64.


#### Output Raw CSV data
The orchestrator and worker pods run independently of the initiator script, with the orchestrator pod sending work items to workers till the testcase schedule is complete. The iperf output (both TPC and UDP modes) and the netperf TCP output from all worker nodes is uploaded to the orchestrator pod where it is filtered and the results are written to the output file as well as to stdout log. Default file locations are `/tmp/result.csv` and `/tmp/output.txt` for the raw results.

**All units in the csv file are in Gbits/second**
```console
ALL TESTCASES AND MSS RANGES COMPLETE - GENERATING CSV OUTPUT
the output for each MSS testpoint is a single value in Gbits/sec 
MSS , Maximum, 96, 160, 224, 288, 352, 416, 480, 544, 608, 672, 736, 800, 864, 928, 992, 1056, 1120, 1184, 1248, 1312, 1376, 1460
1 iperf TCP. Same VM using Pod IP ,24252.000000,22650,23224,24101,23724,23532,23092,23431,24102,24072,23431,23871,23897,23275,23146,23535,24252,23662,22133,,23514,23796,24008,
2 iperf TCP. Same VM using Virtual IP ,26052.000000,26052,0,25382,23702,0,22703,22549,0,23085,22074,0,22366,23516,0,23059,22991,0,23231,22603,0,23255,23605,
3 iperf TCP. Remote VM using Pod IP ,910.000000,239,426,550,663,708,742,769,792,811,825,838,849,859,866,874,883,888,894,898,903,907,910,
4 iperf TCP. Remote VM using Virtual IP ,906.000000,0,434,546,0,708,744,0,791,811,0,837,849,0,868,875,0,888,892,0,903,906,0,
5 iperf TCP. Hairpin Pod to own Virtual IP ,23493.000000,22798,21629,0,22159,21132,0,22900,21816,0,21775,21425,0,22172,21611,21869,22865,22003,22562,23493,22684,217872,
6 iperf UDP. Same VM using Pod IP ,6647.000000,6647,
7 iperf UDP. Same VM using Virtual IP ,6554.000000,6554,
8 iperf UDP. Remote VM using Pod IP ,1877.000000,1877,
9 iperf UDP. Remote VM using Virtual IP ,1695.000000,1695,
10 netperf. Same VM using Pod IP ,7003.430000,7003.43,
11 netperf. Same VM using Virtual IP ,0.000000,0.00,
12 netperf. Remote VM using Pod IP ,908.460000,908.46,
13 netperf. Remote VM using Virtual IP ,0.000000,0.00,
END CSV DATA
```

##### Configuration

The following table lists the configurable parameters of the chart and their default values.

Parameter | Description | Default
--------- | ----------- | -------
`imagePullPolicy` | Whether to Always pull imaged or only IfNotPresent | IfNotPresent
`images.orchestratorVersion` | Image version for the orchestrator | 1.1
`images.workerVersion` | Image version for the worker | 1.1
`debug.orchestrator` | Debug mode for the orchestrator  | false
`debug.worker` | Debug mode for the worker| false
