# Evaluation of Resource Allocation Failure #
In this evaluation, we analyze the behavior of the `KubeAdaptor` in a failure situation of resource allocation.
This situation means that our `ARAS` allocates resource quotas less than `$min_{mem}+\beta$` through the resource
scaling method against a high-concurrency scenario.
So the task pods cannot smoothly execute and turn to `OOMKilled` status due to insufficient memory resources.
Accordingly, the `OOMKilled` task pods make the workflow running get stuck.

We welcome you to download, learn, and work together to maintain the `KubeAdaptor` with us. If you use it for scientific research and
engineering applications, please be sure to protect the copyright and indicate authors and source.
If you have any questions during the operation, please do not hesitate to contact `uzz_scg@163.com`.


In the following, we investigate how KuberAdaptor responds to `OOMKilled` task pods, reallocates resources to execute task pods,
and resumes workflow execution under our `ARAS`.
For this evaluation, we inject `10` Montage workflows into our K8s cluster at a time under the constant arrival pattern.
We fine-tune `$min_{cpu}$` and `$min_{mem}$` to be less than the amount of memory required by the Stress tool in the task pod.
In this evaluation, the minimum memory for a task pod to run, i.e., the amount of memory operated 
by the `Stress` tool in the task pod id set to `2000Mi`.
Herein, we only focus on memory resources because memory resources are incompressible resources, and insufficient memory resources will
trigger the task pod `OOMKilled`, while CPU resources as compressible resources do not.
Subsequently, our `ARAS` tries to reduce the allocated resource quota by the resource scaling method
in response to continuous workflow requests.
When the allocated resource is less than `$min_{mem}+\beta$`(i.e., $2000Mi+20Mi$), `OOMKilled` task pods will appear due to running resource shortage.
KubeAdaptor equipped with our ARAS in this paper can watch `OOMKilled` events, delete these
`OOMKilled` task pods, reallocate computational resources, and regenerate these `OOMKilled` task pods, ensuring continuous execution of workflows.

## Resource description

#### Software Prerequisites

1. OS version: Ubuntu 20.4/CentOS Linux release 7.8
2. Kubernetes: v1.18.6/v1.19.6
3. Docker: 18.09.6.

Note that all docker images in the following `YAML` file are publicly available on DockerHub.
In order to avoid the influence of docker image download delay,
all Docker images need to be downloaded to each local cluster node in advance.

In addition, you need to deploy the `NFS server service` into your cluster in advance so that each node is able to
mount the Master node's shared directory.
Each node equips with an 8-core AMD EPYC 7742 2.2GHz CPU and 16GB of RAM, running Ubuntu 20.4 and K8s v1.19.6 and Docker version 18.09.6.
The Redis database~v5.0.7 is installed on the Mater node.

### ./TaskContainerBuilder

This directory includes the source codes of `KubeAdaptor` with `ARAS`.
You can build the Docker image by the `Dockerfile` file or pull the image of this module from Docker Hub. 
The image `task-container-builder:v10.0` contains `OOM` test functionality.
```console
docker pull shanchenggang/task-container-builder:v10.0
```

#### ./TaskContainerBuilder/informer
The `informer.go` includes detection function of `OOMKilled` pod, 
followed by the deleting of this \verb|OOMKilled| task pod.

### ./experiment/
The directory `./deploy` includes the Yaml files corresponding to `KubeAdaptor`, `RBAC`, `resource usage rate`, `Nfs`, and `workflow injection module`.
We use the `Configmap` method in `Yaml` file to inject workflow information (dependency.json) into the container of the `workflow injection module`.
Refer to `./deploy/Montage-OOM-test.yaml.bak` for details.
Herein, the `edit.sh` takes care of edit these system configuration files.
The `Montage-OOM-test.yaml.bak` contain the definitions of system core functionality pods and four scientific workflows.

**steps:**

* Update the K8s nodes' ip.

  Update the `ipNode.txt` in line with your K8s cluster.

* Update `./deploy/edit.sh`.
  
  Update the `edit.sh` and add `cp Montage-OOM-test.yaml.bak workflowInjector-Builder.yaml`.
  
* Update `./deploy/Montage-OOM-test.yaml.bak`.
  Take `Montage.yaml.bak` as example, you need modify the image address of task-container-builder pod
  and workflow-injector pod according to experimental requirements.
  ```console
  containers:
    - name: task-container-builder-ctr
      #v10.0 ARAS-OOM test 
      image: shanchenggang/task-container-builder:v10.0
  ```
  As shown above, you can set the image address of `task-container-builder` in the light of the `OOM-test` (v10.0).
  
```console
  containers:
    - name: workflow-injector-ctr
      #Constant v9.0, Linear v9.1, Pyramid v9.2
      image: shanchenggang/workflow-injector:v9.0
      imagePullPolicy: IfNotPresent
      #imagePullPolicy: Always
      command: [ "./workflowInjector" ]
      #total number of workflows
      args: [ "10" ]
  ```
  As for the image address of `workflow injector` pod, you adopt the constant workflow arrival pattern.
  The parameter `args` is the number of workflow requests injected by the users.
    ```console
    data:
      #task number in one workflow
      task.numbers: "21"
      redis.server: 0.0.0.0
      redis.port: "6379"
      #quantity per batch in Constant Arrival Pattern
      batch.num: "10"
      #time interval (seconds)
      interval.time: "300"
    ```
  The `task.numbers` indicates the total task number of such workflow type, and `batch.num` indicates the number
  of workflow tasks in each batch. The `interval.time` indicates the interval of each batch.
  The `slope.value` and `initial.value` respectively represent the growth value of each batch and initial value
  under workflow linear arrival and workflow pyramid arrival patterns.

  You can define tasks in your workflow as shown below.
    ```console
    data:
      dependency.json: |
        {
          "0": {
              "input": [],
              "output": ["1","2","3","4"],
              "image": ["shanchenggang/task-emulator:latest"],
              "cpuNum": ["2000"],
              "memNum": ["4000"],
              "args": ["-c","1","-m","2000","-i","3"],
              "labelName": ["app"],
              "labelValue": ["task"],
              "timeout": ["5"],
              "minCpu": ["500"],
              "minMem": ["500"]
          },
         "1": {
             "input": ["0"],
             "output": ["5","6","13"],
             "image": ["shanchenggang/task-emulator:latest"],
             "cpuNum": ["2000"],
             "memNum": ["4000"],
             "args": ["-c","1","-m","2000","-i","3"],
             "labelName": ["app"],
             "labelValue": ["task"],
             "timeout": ["6"],
             "minCpu": ["500"],
             "minMem": ["500"]
         },
       ...
    ```
  Herein, `input` and `output` respectively indicate the ancestor and descendant tasks of the current task.
  `image` represents the Docker Image address of this workflow task.
  The `cpuNum` is the amount of CPU Milli cores required by the users, and `memNum` is the amount of memory capacity
  required by the users.
  The `minCpu` and `minMem` represent a minimum of CPU and memory resources required to run the current task container, respectively.

* Run `./deploy.sh`

  Deploy the `KubeAdaptor` into the K8s cluster. The `./deploy/edit.sh` file firstly captures the Master's IP,
  updates the other corresponding files. Then it copies `Montage-OOM-test.yaml.bak` to `workflowInjector-Builder.yaml`.
  The `deploy.sh` file includes a series of `Kubectl` commands.
  During the workflow lifecycle, you can watch the execution states of workflow tasks. The following is the operation command.
  ```console
  kubectl get pods -A --watch -o wide
  ```
* Run `./clear.sh`

  When the workflow is completed, you can run the `./clear.sh` file to clean up the workflow information and obtain the log files.

* Run `./redisClear.sh`

  Finally, you need to execute `./redisClear.sh` and input `flushall` to clean up the Redis database.
