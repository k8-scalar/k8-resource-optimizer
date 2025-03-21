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
The k8-resource-optimizer is deployed within the k8-cluster. This is preferably done on a seperate node.  This can be achieved by the use of labels and node-selectors ([link](https://kubernetes.io/docs/concepts/overview/working-with-objects/labels/)):
```
$ kubectl label nodes $MY_NODE nodetype=k8-resource
```
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

# Troubleshooting

## Incompatible helm client and server versions

The decomads/k8-resource-optimizer image uses the latest version of the helm client. In case you installed an older version of helm in your cluster you will get the following error:

```

2019/09/08 08:53:24             Bench: intalling  1 injected charts
2019/09/08 08:53:24                     intalling chart 1/1
2019/09/08 08:53:24                     Helmwrap: installing chart: /tmp/k8-reso urce-optimizer/charts/silver-thesisapp-42538 as release: silver-thesisapp-42538
2019/09/08 08:53:24 [install /tmp/k8-resource-optimizer/charts/silver-thesisapp- 42538 --name silver-thesisapp-42538 --namespace silver --wait]
2019/09/08 08:53:25 error in Helmwrap InstallChart: exit status 1, []
2019/09/08 08:53:25             Bench: Deleting 1 Helm charts
2019/09/08 08:53:25                     Helmwrap: deleting release: silver-thesi sapp-42538
2019/09/08 08:53:25 error in Helmwrap DeleteRelease: exit status 1
```
When you run `helm list` you will get the following error: 
```
root@k8-resource-optimizer-59b9f7c9f8-ndm7c:/# helm list
Error: incompatible versions client[v2.14.2] server[v2.8.0]
```
To resolve the problem, you have to install the older helm client inside the container as follows:

```
curl -LO https://kubernetes-helm.storage.googleapis.com/helm-v2.8.0-linux-amd64.tar.gz && tar xvzf helm-v2.8.0-linux-amd64.tar.gz && chmod +x ./linux-amd64/helm && mv ./linux-amd64/helm /usr/local/bin/helm
```

## Enforcing consistency from the Kubernetes scheduler
In order to ensure that Pods are deployed on the same node across multiple tests,  nodeselectors must again used.  For example to ensure that the worker Pods are always deployed on the same node, perform the following 2 steps

1) the node is labeled as follows:
```
$ kubectl label nodes $MY_NODE nodetype=worker
```
2) the template of the worker.yaml file inside the helm chart onder 'charts/thesisapp/templates' must be extended with a nodeSelector property


```
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: worker
  namespace: default
spec:
  replicas: 1
  template:
    metadata:
      ...
    spec:
      ...
      containers:
      - name: worker
        image: ...
        resources:
          requests:
            memory: "..."
            cpu: "..."
           ...
      nodeSelector:
        nodetype: worker
```
# License

Copyright 2018 KU Leuven Research and Development - imec - Distrinet

Licensed under the Apache License, Version 2.0 (the "License"); you may not use this file except in compliance with the License. You may obtain a copy of the License at http://www.apache.org/licenses/LICENSE-2.0

Unless required by applicable law or agreed to in writing, software distributed under the License is distributed on an "AS IS" BASIS, WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied. See the License for the specific language governing permissions and limitations under the License.

Administrative Contact: dnet-project-office@cs.kuleuven.be
Technical Contact: eddy.truyen@cs.kuleuven.be
