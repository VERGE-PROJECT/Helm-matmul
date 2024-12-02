# COMPSs matrix multiplication

URL to download the Chart locally
```
git clone https://gitlab.bsc.es/ppc-bsc/software/compss-matmul-helm.git
```

Installation
```
cd compss-matmul-helm
helm install compss-matmul .
```

## Values file
Make sure the values file adapts to your Kubernetes cluster. By default:
- Image pull policy is set to `Always`
- The master is deployed **without** a volume
- 2 workers with 4 CPU and 4 RAM
- The matmul will be executed 100 times
- The Prometheus Pushgateway svc is `prometheus-prometheus-pushgateway`

<!--
#### Master command
The COMPSs master needs two files in order to be able to function correctly:
1. `resources.xml`. Provides information about all the available resources that can be used for an execution. That is, specifies the architecture, cpu, memory, network adaptor, etc for each worker. 
2. `project.xml`. Provides information about the resources used in a specific execution. That is, lists the workers. 

These two files are generated in the master command, before executing `runcompss`, which will use them. This means the CPU and memory of the workers is specified statically in the resources file, and it is not taken from the workers Kubernetes YAML definition file. So, even though the worker YAML definition is changed, the worker's information kept by the COMPSs master will not be updated automatically. 
-->

## Namespace
If you want to deploy the application in a custom namespace, you have to specify it when executing the `helm install`, such that:
```
helm install compss-matmul --namespace <your-namespace> .
```
If no namespace is specified, the application will run in the `default` namespace.

## COMPSs 
### Master volume
You can deploy the COMPSs master with a local volume attached to it, so its results can be retreived later. The Persistent Volume deployed uses the Storage Class `local-storage`, which does not allow for dynamic provisioning, so the PV and the PVC are created in the deployment. The Storage Class needs to be created before installing the Chart. 
```
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: local-storage
provisioner: kubernetes.io/no-provisioner
volumeBindingMode: WaitForFirstConsumer
```

As for the `compss.master.volume` of the master, you have to check:
1. The `compss.master.volume.localPath` exists locally on the `compss.master.volume.node`.
2. The `compss.master.volume.node` is a the name of one of the nodes of your cluster (you can check with `kubectl get nodes`). Kubernetes will deploy the master pod in the node specified. 

### Matmul parameters
The matmul application takes four parameters:
1. `b`. Number of blocks each matrix will have. **The larger this parameter is, the greater the number of independent tasks available for computation, thereby increasing parallelism.**
2. `e`. Number of elements each block will have. **The larger the value of `e`, the more time it will take for a task to complete its computation before proceeding to the next task that is free of dependencies.**
3. `iterations`. Number of times the same COMPSs execution will perform the matrix multiplication. 
4. `pushgateway_endpoint`. Endpoint for the application to push the results to Prometheus' Pushgateway in the form of `url:port`. 

Executing `matmul.py -b 4 -e 1024` runs 4 x 4 blocks of 1024 x 1024 elements each block, which will conform matrices of 4096 x 4096 elements.

## Prometheus
Prometheus can be installed using Helm: https://github.com/prometheus-community/helm-charts/tree/main/charts/prometheus

For this deployment, only the server and Pushgateway are needed. 

### Reported metrics
The application reports `matmul_total_time` to the **Prometheus server** through **Pushgateway**, under the job `matmul_job`. This `matmul_total_time` field stores the time taken for the application to complete one iteration of matrix multiplications. This value is updated with each iteration. Take into account that if the iteration of the application runs faster than the **Prometheus server** can retrieve information from the **Pushgateway**, some data may be lost.

