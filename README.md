# k8-resource-optimizer
A framework for black-box SLO tuning of multi-tenant applications in Kubernetes

# Demo version of k8-resource-optimizer
The container image contains a binary-only version of k8-resource-optimizer that is configured with an example to demonstrate how to optimize a multi-tenant job processing application. It cannot be used for optimizing other applications. 

## Preparing your Kubernetes cluster
The k8-resource-optimizer must itself be deployed within the k8-cluster. 
The troubleshooting section is at the end of this description.

### 1. Creating service account for k8-resource-optimizer

Service account for k8-resource-optimizer
```
apiVersion: v1
kind: ServiceAccount
metadata:
  name: k8-resource-optimizer
  namespace: default
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: k8-resource-optimizer-binding
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
  - kind: ServiceAccount
    name: k8-resource-optimizer
    namespace: default
```


## Deploying K8-resource-optimizer
The k8-resource-optimizer is deployed within the k8-cluster. 

Deploy the tool using the following yaml file:
```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: k8-resource-optimizer
  namespace: default
  labels:
    app: k8-resource-optimizer
spec:
  replicas: 1
  selector:
    matchLabels:
      app: k8-resource-optimizer
  template:
    metadata:
      labels:
        app: k8-resource-optimizer
    spec:
      serviceAccountName: k8-resource-optimizer
      containers:
      - name: k8-resource-optimizer
        image: decomads/k8-resource-optimizer:latest      
```

# Using the tool
Connect to your running deployment and start an interactive bash-shell
```
$ kubectl exec -it <name of Pod> -- bash

# Once connected move to the exp folder.  

$ cd exp
```
Run the binary with a *--help* flag, as the tool exp:ects all it's input as flags.  For example
```
$ ./k8-resource-optimizer --help

Usage:
  µ-SLO [flags]

Flags:
  -c, --config string       Path to decomposition config
  -e, --experiment string   Name of the experiment
  -h, --help                help for µ-SLO
  -i, --iterations int16    Number of iterations (default -1)
  -l, --loops int16         Number of loops (runs) per thread (default 1)
  -x, --offline             Set to offline
  -p, --optimizer string    Name of optimizer + arguments
  -o, --output string       Path outputdir
  -r, --previous string     path of previous results
  -s, --samples int16       Number of samples per iteration (default -1)
  -t, --threads int16       Number of threads optimizing in parallel (default 1)
  -u, --utilfunc string     Name of util func
```

## Running example of running an optimizer

We'll run the tool for an example decomposition file and helm chart in the exp/examples folder. The parameters defined in the examples/decompositions/thesis-app.yaml map to the Helm chart values: examples/charts/thesisapp/values.yaml, which are injected in the chart templates by helm e.g. in frontend template: examples/charts/thesisapp/templates/frontend.yaml. 

The command below will run a single configuration sample picked by the best config algorithm. Run the command and proceeed to the next step during its execution.
```
$ ./k8-resource-optimizer -c examples/decompositions/thesis-app.yaml -s 1 -i 1 -e thesisapp

2019/08/07 09:20:03 BestConfig: initializing with 8 parameters
2019/08/07 09:20:03 Decomposer: starting iteration: 1/1
2019/08/07 09:20:03 		Bench: injecting charts values with chosen configurations
2019/08/07 09:20:03 			injecting chart 1/1

2019/08/07 09:20:03 		Bench: intalling  1 injected charts
2019/08/07 09:20:03 			intalling chart 1/1
2019/08/07 09:20:03 			Helmwrap: installing chart: /tmp/k8-resource-optimizer/charts/silver-thesisapp-26494
2019/08/07 09:21:23 		Bench: Installed  1 charts via helm. Initially waiting 120 seconds for setup...
2019/08/07 09:23:25 		Bench: running experiment of type: ThesisLocustExperiment

...

2019/08/07 09:26:43 writing report to test/report.csv...
```
Check the setup of the demo application during benchmark.
```
$ kubectl -n silver get pods

NAME                                READY   STATUS    RESTARTS   AGE
frontend-7f48d9f9c-bc527            0/1     Running   0          21s
silver-thesisapp-26494-rabbitmq-0   0/1     Running   0          21s
worker-66b6958694-t28nn             1/1     Running   0          21s
```

The output of the tool can be found in the test folder. It contains the raw locust data for each sample in a folder "iteration-sample", a readable csv file and a json file. 
```
$ ls test
0-0  report.csv  results.json
```
The results.json file can be past as an argument to be used as a dataset of previously obtained results using the `-r` flag. 

## Supported optimization algorithms
```
func InitializeOptimizer(name string, sla models.SLA, nbOfiterations int, nbOfSamplesPerIteration int) (Optimizer, error) {
	switch optimizer := name; optimizer {
	case "bestconfig":
		return bestconfig.CreateBestConfigOptimzer(sla, nbOfiterations, nbOfSamplesPerIteration), nil
	case "bestconfigcontstraint":
		return bestconfigconstraint.CreateBestConfigConstraintOptimzer(sla, nbOfiterations, nbOfSamplesPerIteration), nil
	case "bayesianoptgo":
		return bayesianoptgo.CreateBayesianOptimzer(sla, nbOfiterations, nbOfSamplesPerIteration), nil
	case "exhaustive":
		return exhaustive.CreateExhaustiveSearch(sla), nil
	case "bayesianopt":
		return bayesianopt.CreateBayesianOptimzer(sla, nbOfiterations, nbOfSamplesPerIteration), nil
	case "random":
		return random.CreateRandomSearch(sla), nil
	case "randomincrease":
		return randomincrease.CreateRandomIncrease(sla), nil
	case "MOAT":
		return elementaryeffects.CreateElementaryEffects(sla, nbOfiterations, nbOfSamplesPerIteration), nil
	case "moathist":
		return moatfromhist.CreateElementaryEffects(sla, nbOfiterations, nbOfSamplesPerIteration), nil
	case "bayesianoptfmfn":
		return bayesianoptfmfn.CreateBayesianOptimzer(sla, nbOfiterations, nbOfSamplesPerIteration), nil
	case "heapster":
		return plotheapster.CreatePlotter(sla, nbOfiterations, nbOfSamplesPerIteration), nil
	default:
		log.Fatal("Optimizer: initializeOptimizer: unknown optimizer specified: %v", name)

	}
	return bestconfig.CreateBestConfigOptimzer(sla, nbOfiterations, nbOfSamplesPerIteration), errors.New("Optimizer: initializeOptimizer: unknown optimizer specified")
}
```
## Meaning of --iterations (-i) & --samples (-s) for various optimizers

* ``MOAT`` (screening): Implements morris one-at-a-time elementary effects method.  Iterations map on number of trajectories (-i), samples (-s) map to the levels. The levels are used to calculate the delta between two settings of a parameter. In each trajectory all parameters are altered once. Typically 10 trajectories are used. 

* ``BestConfig``: Iterations are use to find a search space that has a high probability to contain the optimal configuration. s samples are taken per iteration. In each iteration the searchspace is narrowed or a backtracking is performed.

* ``Random``: Iterations set the amout of samples tested. (sample parameter has no effect)

* ``BayesianFmfn``: Iterations are sample configuations tested by bayesian optimization.

* ``Exhaustive``: samples are calculated based on the total possible combinations of parameters. 

## Implemented utility functions
```
func InitializeUtilityFunc(name string) UtilityFunc {
	switch name {
	case "resourceBased":
		return ResourceBasedUtilityFunc
	case "sockshop3":
		return sockshop.ResourceBasedUtilityFunc3Parameters
	case "thesisapp":
		return thesisapp.ResourceBasedUtilityFunc
	case "singlecomponent":
		return SingleComponentUtilityFunc
	case "teastore":
		return TeastoreUtilityFunc
	case "thesisappSLO":
		return thesisapp.IdentityUtilityFuncThesis
	case "teastoreSLO":
		return teastore.IdentityUtilityFuncTeastore
	default:
		log.Panicf("InitializeUtilityFunc: unknown function %v", name)
		return nil
	}
```

# Performance optimization of TeaStore microservices application

To run bayesianoptimization algorithm on the Teastore microservices application for one iteration, execute the following command in the k8-resource-optimizer container:

```
root@k8-resource-optimizer-8587d8c7c9-kmj5g:/exp# ./k8-resource-optimizer -c examples/decompositions/teastore.yaml  -e teastore -i 1
```

Here is the output:

```
2025/05/07 10:17:45
2025/05/07 10:17:45 executing :false
2025/05/07 10:17:45 Decomposer: starting iteration: 1/1
2025/05/07 10:17:45 >>>removing duplicate logs
2025/05/07 10:17:45 >>> end removing duplicate logs
2025/05/07 10:17:45 blocking read
2025/05/07 10:17:45 finish read: 483, <nil>
2025/05/07 10:17:45 Got: {"registryCpu": 612.3777971729004, "dbMemory": 823.2238964275989, "webuiCpu": 651.3774129518425, "authCpu": 836.3283392036175, "webuiMemory": 882.4941723960374, "persistenceCpu": 692.2108349139527, "registryMemory": 1049.8304126710254, "persistenceMemory": 1069.6243243625013, "recommenderMemory": 907.9430050326521, "dbCpu": 1113.792084446537, "recommenderCpu": 1121.2246987703388, "imageCpu": 1104.6407348188236, "imageMemory": 1052.6469655212322, "authMemory": 875.8793295434833}

2025/05/07 10:17:45 map[authCpu:836.3283392036175 authMemory:875.8793295434833 dbCpu:1113.792084446537 dbMemory:823.2238964275989 imageCpu:1104.6407348188236 imageMemory:1052.6469655212322 persistenceCpu:692.2108349139527 persistenceMemory:1069.6243243625013 recommenderCpu:1121.2246987703388 recommenderMemory:907.9430050326521 registryCpu:612.3777971729004 registryMemory:1049.8304126710254 webuiCpu:651.3774129518425 webuiMemory:882.4941723960374]
2025/05/07 10:17:45 map[authCpu:836.3283392036175 authMemory:875.8793295434833 dbCpu:1113.792084446537 dbMemory:823.2238964275989 imageCpu:1104.6407348188236 imageMemory:1052.6469655212322 persistenceCpu:692.2108349139527 persistenceMemory:1069.6243243625013 recommenderCpu:1121.2246987703388 recommenderMemory:907.9430050326521 registryCpu:612.3777971729004 registryMemory:1049.8304126710254 webuiCpu:651.3774129518425 webuiMemory:882.4941723960374]
2025/05/07 10:17:45     Sample: 1/1
2025/05/07 10:17:45             Benchmarking sample: 1/1
2025/05/07 10:17:45             Bench: injecting charts values with chosen configurations
2025/05/07 10:17:45                     injecting chart 1/1
2025/05/07 10:17:45 map[authCpu:500m authMemory:1000Mi authReplicas:1 dbCpu:500m dbMemory:1000Mi dbReplicas:1 image:map[pullPolicy:IfNotPresent] imageCpu:500m imageMemory:1000Mi imageReplicas:1 persistenceCpu:500m persistenceMemory:1000Mi persistenceReplicas:1 recommenderCpu:500m recommenderMemory:1000Mi recommenderReplicas:1 registryCpu:500m registryMemory:1000Mi registryReplicas:1 version:1.2.0 webuiCpu:500m webuiMemory:1000Mi webuiReplicas:1]
2025/05/07 10:17:45 map[authCpu:875m authMemory:896Mi authReplicas:1 dbCpu:1125m dbMemory:768Mi dbReplicas:1 image:map[pullPolicy:IfNotPresent] imageCpu:1125m imageMemory:1024Mi imageReplicas:1 namespace:silver persistenceCpu:750m persistenceMemory:1024Mi persistenceReplicas:1 recommenderCpu:1125m recommenderMemory:896Mi recommenderReplicas:1 registryCpu:625m registryMemory:1024Mi registryReplicas:1 version:1.2.0 webuiCpu:625m webuiMemory:896Mi webuiReplicas:1]
2025/05/07 10:17:45             Bench: intalling  1 injected charts
2025/05/07 10:17:45                     intalling chart 1/1
2025/05/07 10:17:45                     Helmwrap: installing chart: /tmp/k8-resource-optimizer/charts/silver-teastore-58690 as release: silver-teastore-58690
2025/05/07 10:17:45 [install silver-teastore-58690 /tmp/k8-resource-optimizer/charts/silver-teastore-58690 -n silver --wait]
2025/05/07 10:17:49             Bench: Installed  1 charts via helm. Initially waiting 60 seconds for setup...
2025/05/07 10:18:49             Bench: running experiment of type: TeastoreExperiment
2025/05/07 10:18:49             Locust: running with parameters: [/tmp/locustwrap/singleRunAndParse.sh /tmp/sockloadtest/scripttemplate.py teastore/0-0/warmup 10 1 120 /tmp/locustwrap/parser.py http://teastore-webui.silver.svc.cluster.local:8080]
2025/05/07 10:20:50             Locust: running with parameters: [/tmp/locustwrap/singleRunAndParse.sh /tmp/sockloadtest/scripttemplate.py teastore/0-0/results 10 1 240 /tmp/locustwrap/parser.py http://teastore-webui.silver.svc.cluster.local:8080]
2025/05/07 10:24:50             Bench: Deleting 1 Helm charts
2025/05/07 10:24:50                     Helmwrap: deleting release: silver-teastore-58690
2025/05/07 10:25:23             Bench: Deleted 1 Helm charts
2025/05/07 10:25:23             Processing results of sample: 1/1
2025/05/07 10:25:23 slisDist.Amount == 96, slisRequests.Failures 35,out 129000
2025/05/07 10:25:23 teastore  SLO: req: 1000 got: 129000
2025/05/07 10:25:23 slisDist.Amount == 96, slisRequests.Failures 35,out 129000
2025/05/07 10:25:23
configsauthCpu  authMemory      dbCpu   dbMemory        imageCpu        imageMemory     persistenceCpu  persistenceMemory       recommenderCpu  recommenderMemory       registryCpu     registryMemory  webuiCpu webuiMemory      score
0       875     896     1125    768     1125    1024    750     1024    1125    896     625     1024    625     896     -129
2025/05/07 10:25:23 writing report to teastore/report.csv...
```

Inspect the result:


```
root@k8-resource-optimizer-8587d8c7c9-kmj5g:/exp# more teastore/report.csv
config #        authCpu authMemory      dbCpu   dbMemory        imageCpu        imageMemory     persistenceCpu  persistenceMemory       recommenderCpu  recommenderMemory       registryCpu     registryMemory  we
buiCpu  webuiMemory     score   best score      Name    # reqs  # fails Avg      Min    Max     Median  req/s   Name    # reqs  50      66      75      80      90      95      98      99      100
0       875     896     1125    768     1125    1024    750     1024    1125    896     625     1024    625     896     -129    -100    Total   96      35      22861   14      129232  3700    0.4     Total   96
        3800    5000    35000   43000   129000  129000  129000  129000  129000
```

To run the same command for 2 iterations, but using results from the previous run, run the following command:


```
root@k8-resource-optimizer-8587d8c7c9-kmj5g:/exp# ./k8-resource-optimizer -c examples/decompositions/teastore.yaml  -e teastore -i 2 -r teastore/results.json
```

Now you see that the previous results is taken from the `teastore/results.json` file

2025/05/07 10:26:10
2025/05/07 10:26:10 Decomposer loaded dataset containing 1 previously executed benchmarks
2025/05/07 10:26:10 executing :false
2025/05/07 10:26:10 Decomposer: starting iteration: 1/2
2025/05/07 10:26:10 >>>removing duplicate logs
2025/05/07 10:26:10 >>> end removing duplicate logs
2025/05/07 10:26:10 blocking read
2025/05/07 10:26:11 finish read: 483, <nil>
2025/05/07 10:26:11 Got: {"imageMemory": 1052.6469655212322, "registryMemory": 1049.8304126710254, "webuiMemory": 882.4941723960374, "persistenceMemory": 1069.6243243625013, "dbCpu": 1113.792084446537, "dbMemory": 823.2238964275989, "persistenceCpu": 692.2108349139527, "recommenderMemory": 907.9430050326521, "imageCpu": 1104.6407348188236, "registryCpu": 612.3777971729004, "webuiCpu": 651.3774129518425, "authMemory": 875.8793295434833, "authCpu": 836.3283392036175, "recommenderCpu": 1121.2246987703388}

2025/05/07 10:26:11 map[authCpu:836.3283392036175 authMemory:875.8793295434833 dbCpu:1113.792084446537 dbMemory:823.2238964275989 imageCpu:1104.6407348188236 imageMemory:1052.6469655212322 persistenceCpu:692.2108349139527 persistenceMemory:1069.6243243625013 recommenderCpu:1121.2246987703388 recommenderMemory:907.9430050326521 registryCpu:612.3777971729004 registryMemory:1049.8304126710254 webuiCpu:651.3774129518425 webuiMemory:882.4941723960374]
2025/05/07 10:26:11 map[authCpu:836.3283392036175 authMemory:875.8793295434833 dbCpu:1113.792084446537 dbMemory:823.2238964275989 imageCpu:1104.6407348188236 imageMemory:1052.6469655212322 persistenceCpu:692.2108349139527 persistenceMemory:1069.6243243625013 recommenderCpu:1121.2246987703388 recommenderMemory:907.9430050326521 registryCpu:612.3777971729004 registryMemory:1049.8304126710254 webuiCpu:651.3774129518425 webuiMemory:882.4941723960374]
2025/05/07 10:26:11     Sample: 1/1
2025/05/07 10:26:11             Benchmarking sample: 1/1
2025/05/07 10:26:11              Found same previously executed benchmark, returning cached results: .
2025/05/07 10:26:11             Processing results of sample: 1/1
2025/05/07 10:26:11 slisDist.Amount == 96, slisRequests.Failures 35,out 129000
2025/05/07 10:26:11 teastore  SLO: req: 1000 got: 129000
2025/05/07 10:26:11 slisDist.Amount == 96, slisRequests.Failures 35,out 129000
2025/05/07 10:26:11
```

The second iteration notes that not all pods are ready after the installation of the helm chart, waits for 5 seconds and checks again. This cycle is repeated for 120 seconds. 

```
configsauthCpu  authMemory      dbCpu   dbMemory        imageCpu        imageMemory     persistenceCpu  persistenceMemory       recommenderCpu  recommenderMemory       registryCpu     registryMemory  webuiCpu webuiMemory      score
0       875     896     1125    768     1125    1024    750     1024    1125    896     625     1024    625     896     -129
2025/05/07 10:26:11 Decomposer: starting iteration: 2/2
2025/05/07 10:26:11 >>>removing duplicate logs
2025/05/07 10:26:11 >>> end removing duplicate logs
2025/05/07 10:26:11 blocking read
2025/05/07 10:26:11 finish read: 481, <nil>
2025/05/07 10:26:11 Got: {"imageMemory": 822.9040337699435, "registryMemory": 929.7850623533566, "webuiMemory": 1135.1713212100726, "persistenceMemory": 882.9548898683862, "dbCpu": 668.2514619228505, "dbMemory": 1058.2627309726288, "persistenceCpu": 1049.5977893057461, "recommenderMemory": 783.585861399253, "imageCpu": 999.5769629551703, "registryCpu": 835.8433613644893, "webuiCpu": 1075.1941141337466, "authMemory": 908.5190788548283, "authCpu": 900.3035903884804, "recommenderCpu": 555.0579635062654}

2025/05/07 10:26:11 map[authCpu:900.3035903884804 authMemory:908.5190788548283 dbCpu:668.2514619228505 dbMemory:1058.2627309726288 imageCpu:999.5769629551703 imageMemory:822.9040337699435 persistenceCpu:1049.5977893057461 persistenceMemory:882.9548898683862 recommenderCpu:555.0579635062654 recommenderMemory:783.585861399253 registryCpu:835.8433613644893 registryMemory:929.7850623533566 webuiCpu:1075.1941141337466 webuiMemory:1135.1713212100726]
2025/05/07 10:26:11 map[authCpu:900.3035903884804 authMemory:908.5190788548283 dbCpu:668.2514619228505 dbMemory:1058.2627309726288 imageCpu:999.5769629551703 imageMemory:822.9040337699435 persistenceCpu:1049.5977893057461 persistenceMemory:882.9548898683862 recommenderCpu:555.0579635062654 recommenderMemory:783.585861399253 registryCpu:835.8433613644893 registryMemory:929.7850623533566 webuiCpu:1075.1941141337466 webuiMemory:1135.1713212100726]
2025/05/07 10:26:11     Sample: 1/1
2025/05/07 10:26:11             Benchmarking sample: 1/1
2025/05/07 10:26:11             Bench: injecting charts values with chosen configurations
2025/05/07 10:26:11                     injecting chart 1/1
2025/05/07 10:26:11 map[authCpu:500m authMemory:1000Mi authReplicas:1 dbCpu:500m dbMemory:1000Mi dbReplicas:1 image:map[pullPolicy:IfNotPresent] imageCpu:500m imageMemory:1000Mi imageReplicas:1 persistenceCpu:500m persistenceMemory:1000Mi persistenceReplicas:1 recommenderCpu:500m recommenderMemory:1000Mi recommenderReplicas:1 registryCpu:500m registryMemory:1000Mi registryReplicas:1 version:1.2.0 webuiCpu:500m webuiMemory:1000Mi webuiReplicas:1]
2025/05/07 10:26:11 map[authCpu:875m authMemory:896Mi authReplicas:1 dbCpu:625m dbMemory:1024Mi dbReplicas:1 image:map[pullPolicy:IfNotPresent] imageCpu:1000m imageMemory:768Mi imageReplicas:1 namespace:silver persistenceCpu:1000m persistenceMemory:896Mi persistenceReplicas:1 recommenderCpu:500m recommenderMemory:768Mi recommenderReplicas:1 registryCpu:875m registryMemory:896Mi registryReplicas:1 version:1.2.0 webuiCpu:1125m webuiMemory:1152Mi webuiReplicas:1]
2025/05/07 10:26:11             Bench: intalling  1 injected charts
2025/05/07 10:26:11                     intalling chart 1/1
2025/05/07 10:26:11                     Helmwrap: installing chart: /tmp/k8-resource-optimizer/charts/silver-teastore-58690 as release: silver-teastore-58690
2025/05/07 10:26:11 [install silver-teastore-58690 /tmp/k8-resource-optimizer/charts/silver-teastore-58690 -n silver --wait]
2025/05/07 10:26:21             Bench: Installed  1 charts via helm. Initially waiting 60 seconds for setup...
2025/05/07 10:27:21             Bench: not all pods ready of charts, waiting 5 more seconds
2025/05/07 10:27:26             Bench: running experiment of type: TeastoreExperiment
```

2025/05/07 10:27:26             Locust: running with parameters: [/tmp/locustwrap/singleRunAndParse.sh /tmp/sockloadtest/scripttemplate.py teastore/1-0/warmup 10 1 120 /tmp/locustwrap/parser.py http://teastore-webui.silver.svc.cluster.local:8080]
2025/05/07 10:29:27             Locust: running with parameters: [/tmp/locustwrap/singleRunAndParse.sh /tmp/sockloadtest/scripttemplate.py teastore/1-0/results 10 1 240 /tmp/locustwrap/parser.py http://teastore-webui.silver.svc.cluster.local:8080]
2025/05/07 10:33:27             Bench: Deleting 1 Helm charts
2025/05/07 10:33:27                     Helmwrap: deleting release: silver-teastore-58690
2025/05/07 10:34:00             Bench: Deleted 1 Helm charts
2025/05/07 10:34:00             Processing results of sample: 1/1
2025/05/07 10:34:00 slisDist.Amount == 1657, slisRequests.Failures 656,out 11000
2025/05/07 10:34:00 teastore  SLO: req: 1000 got: 11000
2025/05/07 10:34:00 slisDist.Amount == 1657, slisRequests.Failures 656,out 11000
2025/05/07 10:34:00
configsauthCpu  authMemory      dbCpu   dbMemory        imageCpu        imageMemory     persistenceCpu  persistenceMemory       recommenderCpu  recommenderMemory       registryCpu     registryMemory  webuiCpu webuiMemory      score
0       875     896     625     1024    1000    768     1000    896     500     768     875     896     1125    1152    -11
2025/05/07 10:34:00 writing report to teastore/report.csv...
```


# License
Licensed under the Apache License, Version 2.0 (the "License"); you may not use this file except in compliance with the License. You may obtain a copy of the License at http://www.apache.org/licenses/LICENSE-2.0

Unless required by applicable law or agreed to in writing, software distributed under the License is distributed on an "AS IS" BASIS, WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied. See the License for the specific language governing permissions and limitations under the License.

