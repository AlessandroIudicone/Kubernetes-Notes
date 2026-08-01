# CKAD Certification

## Architecture

A Node is a machine (physical or virtual) and can be:

- worker node: where applications run;
- master node: responsible for managing the cluster. Is responsible of the orchestration of the nodes.

A cluster is a set of nodes grouped together.

The components of a Kubernetes cluster are the following:

| component               | on node | description                                                                                                                 |
| ----------------------- | ------- | --------------------------------------------------------------------------------------------------------------------------- |
| kube-apiserver          | master  | (also kube-apiserver) acts as the front-end. Users, CLI, devices talk to this component to interact with the cluster        |
| etcd                    | master  | is a distributed and reliable key-value store used by Kubernetes to store the data to manage the cluster                    |
| kube-scheduler          | master  | distributes workloads across the worker nodes                                                                               |
| kube-controller-manager | master  | the brain behind orchestration, responsible for monitoring the cluster state and reconciling it with the desired state      |
| Container runtime       | worker  | is the underlying software used to run containers (it can be containerd or CRI-O or Docker Engine¹)                         |
| kubelet                 | worker  | the agent that runs on each node in the cluster, which is responsible to ensure that the containers are running as expected |
| kube-proxy              | worker  | configures the networking rules that allow Services to reach the appropriate Pods                                           |

There is also an optional component for the Control Plane, the `cloud-controller-manager`, which Integrates with underlying cloud provider(s).

> [!IMPORTANT]
>
> ¹ Docker is **not** a native Kubernetes container runtime anymore.
>
> Until Kubernetes 1.23, the kubelet interacted directly with the Docker Engine through a component called `dockershim`.  
> Since Kubernetes v1.24, the built-in `dockershim` has been removed.
>
> Nowadays, the Docker Engine can be used only through a CRI adapter, called cri-dockerd, developed and maintained by Mirantis and Docker.  
> Modern Kubernetes clusters typically use **containerd** or **CRI-O** as the container runtime.
>
> Docker can still be used:
>
> - to build and push container images;
> - as a container runtime only through the external **cri-dockerd** adapter.
>
> ```text
>  <v1.24         >=v1.24
>
> Kubelet        Kubelet
>     │              │
> Dockershim        CRI
>     │              │
> Docker Engine  cri-dockerd
>     │              │
> containerd     Docker Engine
>     │              │
>   runc         containerd
>                    │
>                   runc
> ```

```text
Cluster
│
├── Control Plane
│   ├── API Server
│   ├── etcd
│   ├── Scheduler
│   └── Controller Manager
│
└── Worker Nodes
    ├── kubelet
    ├── kube-proxy
    └── Container runtime
```

Then, then Command Line Tool is the `kubectl` (also called Kube Control).

> [!TIP]
>
> The Control Plane is the software "brain" of a Kubernetes cluster that makes decisions and manages state.  
> The Master Node is the physical or virtual server (or servers) where those Control Plane software components are actually running.  
> Think of the Control Plane as the mind, and the Master Node as the physical body that houses it.

## Various

Useful commands:

- `kubectl cluster-info`: show information about the cluster;
- `kubectl describe`: show details of a specific resource or group of resources;
- `kubectl create -f <file-name>`: creates the resources specified inside the file;
- `kubectl replace --force f <filename>`: deletes and recreates the resources specified inside the file;
- `kubectl api-resources`: lists all resources (especially useful when you can't find a resource on the documentation pages);
- `kubectl explain <resource name>`: gives details on the Kind and lists the top level fields and their type;
- `kubectl explain <resource name>.<field name>`: gives details on the Kind and lists the subfields and their type;
- `kubectl explain <resource name> --recursive`: lists all fields that we would put in the yaml file;
- `kubectl get all --no-headers`: list all the objects created in the default namespace, without printing the header line;
- `kubectl get all -A`: list all the objects in all namespaces;
- `kubectl <command> [subcommand] --help`: gives information about the current command and subcommand, included the different available parameters;
- `kubectl edit <resource> <resource name>` edit manifest of a resource on-the-fly.

The configuration file of a Kubernetes object (derived from a Kind) is called "manifest".

## Pod

Useful commands:

- `kubectl run -n finance redis --image=redis --restart=Never`: create a pod named redis, with the redis image, in the namespace "finance", with restart policy set to `never`;
- `kubectl get pod <pod-name> -o wide`: kubectl get pod but with output in the plain-text format with additional information. For pods, the node name is included;
- `kubectl get pod <pod-name> -o yaml > pod-definition.yaml`: outputs the current configuration of a pod (but can be used for objects in general) to a yaml file;
- `kubectl get pods -w`: list pods and watches for changes;
- `kubectl run httpd --image=httpd:alpine --port=80`: creates an httpd pod and declares port 80 as `containerPort`;
- `kubectl run httpd --image=httpd:alpine --port=80 --expose=true`: in addition to the command above, it automatically also creates a `Service` of type `ClusterIP` that exposes the port 80 inside the cluster, which then adds to DNS, allowing other pods to resolve it with ease.

You should remember the structure of a Pod yaml manifest

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx
spec:
  containers:
  - name: nginx
    image: nginx:1.14.2
    ports:
    - containerPort: 80
```

> [!TIP]
>
> `containerPort` is optional metadata.
>
> It does **not** open a port and Kubernetes does not use it to forward traffic.
>
> The application inside the container decides which port to listen on.
>
> A Service must forward traffic (`targetPort`) to the port where the application is actually listening, regardless of whether `containerPort` is declared.

In a Pod manifest, you can also optionally specify the `command` and `args` fields for each container.  
The first overrides the default command (defined in the image) while the second overrides the arguments of the command.  
For comparison with some container runtimes (like Docker):

- using `command` is like specifying the `[Entrypoint]` in the Dockerfile or `--entrypoint` in CLI;
- using `args` is like specifying the `[CMD]` in the Dockerfile or the arguments passed by the CLI.

`command` overrides the Docker image's `ENTRYPOINT` while `args` overrides the Docker image's `CMD`.  
If only `args` is specified, Kubernetes keeps the image's ENTRYPOINT and replaces only the CMD.  
If `command` is specified, the image's ENTRYPOINT is ignored.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: command-demo
  labels:
    purpose: demonstrate-command
spec:
  containers:
  - name: command-demo-container
    image: debian
    command: ["printenv"]
    args: ["HOSTNAME", "KUBERNETES_PORT"]
  restartPolicy: OnFailure
```

The `command` and `args` fields are arrays of strings.  
YAML provides multiple equivalent syntaxes to represent arrays.  
The following examples still produce the same object.

```yaml
    command:
    - printenv
    args:
    - HOSTNAME
    - KUBERNETES_PORT
```

```yaml
    command:
    - "printenv"
    args:
    - "HOSTNAME"
    - "KUBERNETES_PORT"
```

Quotation marks are optional in this case, as they do not affect the meaning. They should only be used when a value might otherwise be interpreted by YAML as a different data type, for example:

- `"true"` is a string, whereas `true` is a boolean;
- `"1200"` is a string, whereas `1200` is a number.

You can also declare a pod to start with custom `command` and `args` with the imperative syntax

```bash
kubectl run nginx --image=nginx --command -- <cmd> <arg1> .. <argn>
```

You can also specify environment variables inside a pod, with the `env` property.  
`env` is an array, so every element starts with a dash indicating that is an item inside the array.  
Each item has a `name` and a `value` properties.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx
spec:
  containers:
  - name: nginx
    image: nginx:1.14.2
    ports:
    - containerPort: 80
    env:
    - name: HOST
      value: my-site.com
    - name: PORT
      value: "80"
    - name: SERVER_NAME
      value: my-site
    - name: DB_PASSWORD
      value: unprotected-password
```

However, there are other ways of setting environment values in Kubernetes, such as injecting them through ConfigMaps and Secrets, which we'll see later.

## ReplicaSet

This is the structure of a `ReplicaSet` yaml manifest

```yaml
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: frontend
  labels:
    app: guestbook
    tier: frontend
spec:
  replicas: 3
  selector:
    matchLabels:
      tier: frontend
  template:
    metadata:
      labels:
        tier: frontend
    spec:
      containers:
      - name: php-redis
        image: us-docker.pkg.dev/google-samples/containers/gke/gb-frontend:v5
```

In the past, the `ReplicationController` (introduced in the initial Kubernetes v1.0 release in July 2015) was used instead of `ReplicaSet` (introduced later in Kubernetes v1.2 in March 2016).
The main difference between the `ReplicationController` (deprecated) and the `ReplicaSet` is that the field `selector` became mandatory in the second (in case omitted, in the former `ReplicationController`, Kubernetes assumes that `selector` is the same as the `labels` field).

In a Kubernetes `ReplicaSet`, the labels defined in `spec.selector.matchLabels` must exactly match the labels defined in `spec.template.metadata.labels`.  
If they do not match, the Kubernetes API server will reject the deployment and return an error.

Some commands to remember:

- `kubectl create -f replicaset-definition.yml`: creates ReplicaSet starting from the definition inside the yaml manifest;
- `kubectl get replicaset <replicasetname>`: retrieves the ReplicaSet in the default namespace;
- `kubectl delete replicaset <replicasetname>`: deletes a replicaset and all of its pods;
- `kubectl replace -f replicaset-definition.yml`: replaces or updates a ReplicaSet;
- `kubectl scale`: scales the replicas of a ReplicaSet.

To scale a ReplicaSet with the command `scale`, there are three options:

- `kubectl apply -f replicaset-definition.yml`: modify the value of `replicas` inside the file manifest and then `apply` it;
- `kubectl scale --replicas=6 -f replicaset-definition.yml`: this does not need the modified file (but an input file is still needed) but it does not updates it;
- `kubectl scale --replicas=6 replicaset frontend`: this is an imperative command that scales the already deployed ReplicaSet, without the need of a manifest file.

There are also options to automatically scale the replicas based on the load, but we'll see them later on.

> [!WARNING]
>
> Updating the Pod template of a ReplicaSet does not trigger an automatic rolling update.
>
> The updated Pod template is used only for newly created Pods.
>
> Existing Pods are not automatically replaced and continue running with their current configuration until they are deleted.
>
> Deployments should be preferred when controlled rolling updates or rollbacks are required.

## Deployment

The `Deployment` is "higher in hierarchy" compared to `ReplicaSet` and `ReplicationController`.  
If the RS and the RC are, in some way, pod containers, a Deployment manages one or more ReplicaSets, providing rollout and rollback capabilities.

The structure of a `Deployment` yaml manifest is almost identical to the one of the `ReplicaSet`, except for the Kind.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  labels:
    app: nginx
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.14.2
        ports:
        - containerPort: 80
```

At creation, the Deployment automatically generates also a `ReplicaSet` (which will be named after the `Deployment`) along obviously with the `Deployment` object.

Useful commands:

- `kubectl scale deployment nginx --replicas=4`: scale a deployment using the kubectl scale command;
- `kubectl create deployment <deployment name> --image=<image>`: create a `Deployment`;
- `kubectl get deployment <deployment name>`: retrieves a deployment based on its name;
- `kubectl get deployments`: retrieves all deployments from the default namespace;
- `kubectl edit deployment <deployment name>`: edit a property of a deployment.

> [!IMPORTANT]
>
> Remember, that you cannot edit specifications of an existing Pod other than:
>
> - `spec.containers[*].image`
> - `spec.initContainers[*].image`
> - `spec.activeDeadlineSeconds`
> - `spec.tolerations`
>
> If you really need, you can instead recreate the pod with `kubectl replace --force f <filename>`.
>
> With Deployments, instead, you can easily edit any field / property of the POD template.  
> Since the pod template is a child of the deployment specification, with every change the deployment will automatically delete and create a new pod with the new changes.  
> So if you are asked to edit a property of a POD part of a deployment you may do that simply by running the command `kubectl edit deployment <deployment name>`.

## Namespace

The `Namespace` is the logical segmentation of Kubernetes clusters.  
Many kinds (like the pods) only exists inside a namespace and in this case we say that the resource is **namespaced**; other resources are not limited to namespaces, like namespaces itself (they are reachable from the whole clusters).  
Each namespace can have its own set of policies defining who can do what, their quotas (n CPU, GB of RAM), limits, etc.

When a Kubernetes cluster is created, it creates:

- the default namespace, which is normally named `default`;
- a system namespace named `kube-system`, in which a series of pods and services are created for serving the cluster, isolating them to the user and preventing that they are modified or deleted;
- a third namespace created automatically is named `kube-public`, which is readable by all users, including unauthenticated users in many cluster configurations (this namespace is a good place to put resources that should be available to all the users).

Inside the same namespace, the resources can refer each other simply using their service name (like if they are people living inside the same house). In this way, for example, the `pod` named `webapp` can reach the `service` named `db-service` which resides in the same namespace simply using its name, like for example with `mysql.connect("db-service")`.  
Instead, if an application would like to reach a service in the same cluster but in another namespace, it should use the syntax `<Service Name>.<Namespace>.svc.cluster.local`, so for example, in case it wants to reach a mysql db on the service `db-service` in the namespace `dev`, it should use the syntax `mysql.connect("db-service.dev.svc.cluster.local")`.  
This happens because:

- when a service is created, a DNS entry is automatically added;
- `cluster.local` is the default domain of a Kubernetes cluster;
- `svc` stands for `Service`.

The `namespace` can also be specified inside the `metadata` section of the various manifest files described above.  
There is also an option to show the resources in all namespaces, which is `--all-namespaces` and the shorthand is `-A`.

Some commands:

- `kubectl get pod --all-namespaces`: retrieves all the pods in all the namespaces;
- `kubectl get service -A`: retrieves all the services in all the namespace;
- `kubectl get service -n [namespace name]`: retrieves all the services in a specific namespace (if not specified, the default namespace will be used).

## Service

Services are Kubernetes objects that allow communication between the workloads and with external entities.  
Services allow loose coupling (abstracting) relatively to the physical addresses of the workloads.  

> [!NOTE]
>
> A Service can select Pods running on different nodes, but Kubernetes does not create one Service per node.
>
> A Service is **NOT** replicated across multiple nodes but is a namespaced Kubernetes object.
>
> Instead, the Service maintains an object called `Endpoints` with a list of endpoints corresponding to all matching Pods, regardless of the node where they run.
>
> Networking components such as kube-proxy (or eBPF-based implementations like Cilium) configure the forwarding rules on the cluster nodes.

There are mainly 3 types of services:

- `ClusterIP`: it creates a virtual IP inside the cluster to enable communication between services. It stays inside the Kubernetes network;
- `NodePort`: exposes a specific port on every node and forwards incoming requests to the selected Pods;
- `LoadBalancer`: provisions a LoadBalancer in supported cloud providers (to distribute the load). Normally is used for exposing the pods on public networks.

Most Services are assigned a virtual IP address called ClusterIP; Headless Services are an exception and are created with `clusterIP: None`.

In Services, the `selector` defines to which pods the service is assigned.  
To link the services with the pods, you need that the content of `selector` in the former equals the definition in `labels` in the latter.  
A Service exposes, through the correct use of the selectors:

- a single pod on a single node;
- multiple pods on a single node;
- multiple pods on multiple nodes.

This happens because each service has a built-in LoadBalancer to distribute the load across different pods.

Example of `Service` of type `ClusterIP`

```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-service
spec:
  selector:
    app.kubernetes.io/name: MyApp
  ports:
    - protocol: TCP
      port: 80
      targetPort: 9376
```

`ClusterIP` is the default type of `service`; if the type is not declared, the Service is automatically created with the `ClusterIP` type.  
Once created, the `ClusterIP` service allows accesses to other pods both using the CLUSTER-IP (number) assigned and its name.

Example of `Service` of type `NodePort`

```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-service
spec:
  type: NodePort
  selector:
    app.kubernetes.io/name: MyApp
  ports:
    - port: 80
      # By default and for convenience, the `targetPort` is set to
      # the same value as the `port` field.
      targetPort: 80
      # Optional field
      # By default and for convenience, the Kubernetes control plane
      # will allocate a port from a range (default: 30000-32767)
      nodePort: 30007
```

In the service of type NodePort, in each element of the array `ports`, along with `port` (port of the service himself) and `targetPort` (destination port on the pod), there is the NodePort, which is the port exposed externally of the Kubernetes cluster.  
The NodePort must always be inside the range 30000 - 32767; it is an optional field and if not declared, it will be assigned an arbitrary value.

Once created this type of service, we can access the pod exposed through `curl http://<node_ip>:<nodePort>`.  

NodePort directly exposes a port on every cluster node.  
The use case of the `NodePort` service are mainly debugging, temporary operations, deployments on non-sensitive environment.  
With NodePort, the client must make the calls to the IP address of one of the nodes and the nodeport. This has two main disadvantage:

- if nodes are replaced / updated / added, they may be assigned a new IP address and this breaks the pattern of the previous calls;
- you declare the IP address of the Kubernetes nodes, and this may raise security concerns.

For production environments it is often preferable to place a LoadBalancer or an Ingress in front of applications to provide more flexibility and traffic management features.  

Example of `Service` of type `LoadBalancer`

```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-service
spec:
  selector:
    app.kubernetes.io/name: MyApp
  ports:
    - protocol: TCP
      port: 80
      targetPort: 9376
  clusterIP: 10.0.171.239
  type: LoadBalancer
status:
  loadBalancer:
    ingress:
    - ip: 192.0.2.127
```

The Service types can be viewed as incremental extensions:

- ClusterIP exposes the Service inside the cluster through a virtual IP;
- NodePort includes all the features of ClusterIP and additionally exposes the Service on a fixed port on every cluster node;
- LoadBalancer extends a NodePort Service by requesting the underlying infrastructure (typically a cloud provider) to provision an external load balancer that forwards traffic to the Service's NodePort.

Sometimes you don't need load-balancing and a single Service IP.  
In this case, you can create a Headless Service, by explicitly specifying

```yaml
spec:
  clusterIP: None
```

With the headless service, DNS queries return the IP addresses of the individual pods behind the Service.  
This is commonly used by StatefulSets.  
So the headless service allows to communicate with an exact pod instead of landing on a random one chosen by the ClusterIP.  
An use case of this type of service, is when you have to deal with a stateful application; for example a Database, when there is a master node that allows writing and many worker nodes that allows only reading, and you need to chose to land on one of them based if you need to write or read data.

For all types of `Service`:

- inside `spec` we define the array `ports`. If we map only one port, the field `name` is optional; if we need to map two or more ports, the parameter `name` becomes mandatory;
- when a `Service` is created, Kubernetes automatically creates also an object called `Endpoints` with the same name of the `Service`, which role is to keep track of which pods are the endpoints of the service. The endpoints are updated when these pod are deleted or updated (like when restarted, scaled, rescheduled or other) in order to target the new created pods to remove an endpoint;
- the `targetPort` should match the port on which the application inside the container is actually listening;
- when creating a `Service`, an IP address (called CLUSTER-IP of the Service) is assigned to it, which allows access from the other workloads both using the CLUSTER-IP (number) assigned or its name.

Here is an example of a `Service` of type `ClusterIP` with two ports, where the field `name` must be present for each port.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-service
spec:
  selector:
    app.kubernetes.io/name: MyApp
  ports:
    - name: http
      protocol: TCP
      port: 80
      targetPort: 9376
    - name: https
      protocol: TCP
      port: 443
      targetPort: 9377
```

Some commands:

- `kubectl get services`: list the services with their status inside the default namespace;
- `kubectl describe service <service name>`: shows detailed information about the service, listing the associated `Endpoints`.

## Imperative commands

While you would be working mostly using the declarative commands (using definition files), imperative commands can help in getting one-time tasks done quickly, as well as generate a definition template easily.  
This would help save a considerable amount of time during your exams.  
Familiarize yourself with the two options that can come in handy while working with the below commands:

- `--dry-run`: by default, as soon as the command is run, the resource will be created. If you simply want to test your command, use the `--dry-run=client` option. This will not create the resource and instead, tell you whether the resource can be created and if your command is right;
- `-o yaml`: this will output the resource definition in YAML format on the screen.

Use the above two in combination along with Linux output redirection to generate a resource definition file quickly, that you can then modify and create resources as required, instead of creating the files from scratch.

Here some examples

| Command                                                      | Description                         | DryDrun command to generate YAML manifest (and eventually redirect it to a file)                |
| ------------------------------------------------------------ | ----------------------------------- | ----------------------------------------------------------------------------------------------- |
| `kubectl run nginx --image=nginx`                            | Create an NGINX Pod                 | `kubectl run nginx --image=nginx --dry-run=client -o yaml > nginx-pod.yaml`                     |
| `kubectl create deployment --image=nginx nginx`              | Create a deployment                 | `kubectl create deployment --image=nginx nginx --dry-run -o yaml`                               |
| `kubectl create deployment nginx --image=nginx --replicas=4` | Create a deployment with 4 Replicas | `kubectl create deployment nginx --image=nginx--dry-run=client -o yaml > nginx-deployment.yaml` |

The following command creates a service named redis-service of type ClusterIP to expose the pod redis on port 6379, via `expose` command

```bash
kubectl expose pod redis --port=6379 --name redis-service --dry-run=client -o yaml
```

and this will automatically use the pod's labels as selectors.  
Or instead, you can do the same via service creation command

```bash
kubectl create service clusterip redis --tcp=6379:6379 --dry-run=client -o yaml
```

but this will not use the pod labels as selectors; instead it will assume selectors as `app=redis`.  
Considering that you cannot pass in selectors as an option, it does not work well if your pod has a different label set.  
So with this second option, after having generated the file, you then need to modify the selectors before creating the service.

For creating a service named nginx of type NodePort to expose pod nginx's port 80 on port 30080 on the nodes, via pod exposition, you need the following command

```bash
kubectl expose pod nginx --port=80 --name nginx-service --type=NodePort --dry-run=client -o yaml
```

to generate the definition file, and considering that:

- this will automatically use the pod's labels as selectors;
- but you cannot specify the node port;

you then need to add the node port in the file manually before creating the service with the pod.  
Or instead, you can do similarly via service creation command

```bash
kubectl create service nodeport nginx --tcp=80:80 --node-port=30080 --dry-run=client -o yaml
```

But this will not use the pods' labels as selectors and you still should then modify the manifest before applying it.

Both the above commands have their own challenges.  
While one of it cannot accept a selector, the other cannot accept a nodeport.  
I would recommend going with the `kubectl expose` command, and then manually input the nodeport in the manifest file before creating the service.

## Formatting Output with kubectl

The default output format for all kubectl commands is the human-readable plain-text format.  
The -o flag allows to output the details in several different formats.

```bash
kubectl [command] [TYPE] [NAME] -o <output_format>
```

Here are some of the commonly used formats:

- `-o json`: output a JSON formatted API object;
- `-o name`: print only the resource name and nothing else;
- `-o wide`: output in the plain-text format with additional information;
- `-o yaml`: output a YAML formatted API object.

## ConfigMaps

Instead of defining environment variables in the pod definition files, we can inject them into the containers of the pods using ConfigMaps.  
ConfigMaps are used to pass configuration data in a form of key-values in Kubernetes.

There are two phases involved in setting ConfigMaps:

1) create ConfigMap;
2) inject into pod.

Like other Kubernetes objects, we can create the CM in two ways:

- imperative: `kubectl create configmap` specifying the required arguments;
- declarative: `kubectl create -f [file name]`.

Imperatively, this can be made by declaring the key values in the `kubectl` command

```bash
kubectl create configmap <configmap name> --from-literal=<key>=<value>
kubectl create configmap app-config --from-literal=APP_COLOR=blue --from-literal=APP_ENV=prod
```

or (still imperatively) though a file (not a manifest)

```bash
kubectl create configmap <configmap name> --from-file=<path-to-file>
kubectl create configmap app-config --from-file=app_config.properties
```

With the declarative approach instead, we do the following

```bash
kubectl create -f <file name>
kubectl create -f config-map.yaml
```

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
data:
  APP_COLOR: blue
  APP_ENV: prod
```

It is important to name the ConfigMaps appropriately as we'll use this names later to associate them with the pods.

Some commands:

- `kubectl get configmaps`: list the configmaps inside the default namespace;
- `kubectl describe configmaps`: also list the configuration data of the configmaps.

In order to inject the configMap, we can use the property `envFrom` on the pod manifest.  
`envFrom` is a list and we can use it to pass as many environment variables as required.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: simple-webapp-color
spec:
  containers:
  - name: simple-webapp-color
    image: simple-webapp-color
    ports:
    - containerPort: 8080
    envFrom:
    - configMapRef:
        name: app-config
```

And then create the pod with `kubectl create -f pod-definition.yaml`.

## Secrets

Secrets are similar ConfigMaps but but are specifically intended to hold confidential data.  
Their data are stored in an encoded or hashed format by default, but it is desirable to encrypt them.

As of configmaps, there are two phases involved in setting Secrets:

1) create Secret;
2) inject into pod.

In addition, there are also different kinds of Secrets, with different use cases and expected keys, on which some checks can be performed by Kubernetes.

| Secret type                         | Typical use                             | Expected keys           |
| ----------------------------------- | --------------------------------------- | ----------------------- |
| Opaque                              | arbitrary user-defined data             | Any keys                |
| kubernetes.io/tls                   | data for a TLS client or server         | `tls.crt`, `tls.key`    |
| kubernetes.io/dockerconfigjson      | serialized `~/.docker/config.json` file | `.dockerconfigjson`     |
| kubernetes.io/dockercfg             | serialized `~/.dockercfg` file          | `.dockercfg`            |
| kubernetes.io/basic-auth            | credentials for basic authentication    | `username`, `password`  |
| kubernetes.io/ssh-auth              | credentials for SSH authentication      | `ssh-privatekey`        |
| bootstrap.kubernetes.io/token       | bootstrap token data                    | many keys expected      |
| kubernetes.io/service-account-token | ServiceAccount token                    | Automatically generated |

`Opaque` is the default Secret type if you don't explicitly specify a type in a Secret manifest.  
When you create a Secret using kubectl, you must use the `generic` subcommand to indicate an `Opaque` Secret type.

And also for the secrets, we can generate them in two ways:

- imperative: `kubectl create secret` specifying the required arguments;
- declarative: `kubectl create -f [file name]`.

Imperatively, this can be made by declaring the key values in the `kubectl` command

```bash
kubectl create secret generic <secret name> --from-literal=<key>=<value>
kubectl create secret generic app-secret --from-literal=DB_Host=mysql --from-literal=DB_User=root --from-literal=DB_Password=paswrd
```

(but this can be complicated when we need to pass many secrets)

or (still imperatively) though a file (not a manifest)

```bash
kubectl create secret generic <secret name> --from-file=<path-to-file>
kubectl create secret generic app-secret --from-file=app_secret.properties
```

With the declarative approach instead, we do the following

```bash
kubectl create -f <file name>
kubectl create -f secret-data.yaml
```

But when using a declarative approach, we must specify the data in `Base64` format.  
For this scope, we can use the `base64` Linux utility `echo -n 'DATA' | base64`:

```console
$ echo -n 'mysql' | base64
bXlzcWw=
$ echo -n 'root' | base64
cm9vdA==
$ echo -n 'paswrd' | base64
cGFzd3Jk
```

which makes the secret definition file like the following

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: app-secret
data:
  DB_Host: bXlzcWw=
  DB_User: cm9vdA==
  DB_Password: cGFzd3Jk
```

> [!CAUTION]
>
> The `data` field stores values encoded in Base64.  
> The `stringData` field accepts plain text values and is converted automatically by the Kubernetes API server in Base64 when the Secret is created or updated and stored into the `data` field.
>
> Base64 is **not encryption**. It is only an encoding format used to represent binary data as text.  
> Kubernetes Secrets are, by default, stored unencrypted in the API server's underlying data store (etcd).  
> In order to safely use Secrets, take at least the following steps:
>
> - enable Encryption at Rest for Secrets;
> - enable or configure RBAC rules with least-privilege access to Secrets;
> - restrict Secret access to specific containers;
> - consider using external Secret store providers (like AWS Provider, Azure Provider, GCP Provider or Vault Provider).

In order to decode the hashed values, use the same `base64` utility adding a `--decode` parameter, so `echo -n 'DATA' | base64 --decode`:

```console
$ echo -n 'bXlzcWw=' | base64 --decode
mysql

$ echo -n 'cm9vdA==' | base64 --decode
root

$ echo -n 'cGFzd3Jk' | base64 --decode
paswrd

```

Some commands:

- `kubectl get secrets`: list the secrets inside the default namespace;
- `kubectl describe secrets`: shows the attributes of the secrets but **hides the values**;
- `kubectl get secret <secret name> -o yaml`: list the attributes of a specific secret, **along with the hashed values**.

In order to inject the secret, also here we can use the property `envFrom` on the pod manifest.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: simple-webapp-color
spec:
  containers:
  - name: simple-webapp-color
    image: simple-webapp-color
    ports:
    - containerPort: 8080
    envFrom:
    - secretRef:
        name: app-secret
```

And then create the pod with `kubectl create -f pod-definition.yaml`.

## Other options to inject ConfigMaps and Secrets

Along with `configMapRef` and `secretRef` that allows to inject all key-values of a resource as environment variables,  
there is also the option `configMapKeyRef` and `secretKeyRef` for choosing the specific keys to import.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx
spec:
  containers:
  - name: nginx
    image: nginx:1.14.2
    ports:
    - containerPort: 80
    env:
    - name: HOST
      valueFrom:
        configMapKeyRef:
          name: nginx-config
          key: nginx.host
    - name: DB_PASSWORD
      valueFrom:
        secretKeyRef:
          name: db-credentials
          key: password
    envFrom:
    - configMapRef:
        name: nginx-config-bulk
    - secretRef:
        name: nginx-secrets-bulk
```

Also, instead of exposing ConfigMaps and Secrets as environment variables, they can also be mounted as volumes.  
In this case, each key becomes a file within the mounted volume with:

- the filename corresponding to the key;
- the file content corresponding to the associated value.

The following example mounts both a ConfigMap and a Secret as Volumes.

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: nginx-config
data:
  nginx.conf: |
    events {}

    http {
      server {
        listen 443 ssl;

        ssl_certificate     /etc/nginx/tls/tls.crt;
        ssl_certificate_key /etc/nginx/tls/tls.key;

        location / {
          return 200 "Hello from Kubernetes!\n";
        }
      }
    }
```

```bash
openssl req \
  -x509 \
  -newkey rsa:2048 \
  -nodes \
  -days 365 \
  -keyout tls.key \
  -out tls.crt \
  -subj "/CN=localhost"
```

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: nginx-tls
type: kubernetes.io/tls
stringData:
  tls.crt: |
    -----BEGIN CERTIFICATE-----
    [replace generated tls.crt content]
    -----END CERTIFICATE-----
  tls.key: |
    -----BEGIN PRIVATE KEY-----
    [replace with generated tls.key content]
    -----END PRIVATE KEY-----
```

> [!NOTE]
>
> Here we used `stringData` for the `tls.crt` and `tls.key` elements of the secret, which allows us to write data in a non-encoded format.  
> When applied, Kubernetes automatically encodes and put it in Base64 format inside the field `data`.
>
> We are also using a Secret of type `kubernetes.io/tls` instead of a generic `opaque` type.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx-with-volumes
spec:
  containers:
  - name: nginx
    image: nginx:1.14.2
    ports:
    - containerPort: 443
    volumeMounts:
    - name: nginx-config
      mountPath: /etc/nginx/nginx.conf
      subPath: nginx.conf
    - name: nginx-tls
      mountPath: /etc/nginx/tls
      readOnly: true
  volumes:
  - name: nginx-config
    configMap:
      name: nginx-config
  - name: nginx-tls
    secret:
      secretName: nginx-tls
```

> [!NOTE]
>
> Here we used `subPath` for the VolumeMount of the configuration, and in this case automatic updates are not performed.
>
> We are also using `readOnly: true`, which is optional for Secret and ConfigMap volumes (they are read-only by nature), but explicitly specifying it makes the intent clearer.

```bash
kubectl apply -f nginx-config.yaml
kubectl apply -f nginx-tls.yaml
kubectl apply -f nginx-with-volumes.yaml
```

And we can see that the files are mounted as expected with the following commands

```console
$ kubectl exec nginx-with-volumes -- cat /etc/nginx/nginx.conf
events {}

http {
  server {
    listen 443 ssl;

    ssl_certificate     /etc/nginx/tls/tls.crt;
    ssl_certificate_key /etc/nginx/tls/tls.key;

    location / {
      return 200 "Hello from Kubernetes!\n";
    }
  }
}

$ kubectl exec nginx-with-volumes -- ls /etc/nginx/tls
tls.crt
tls.key

```

> [!TIP]
>
> This is a common production pattern:
>
> - the ConfigMap stores the NGINX configuration, which can change frequently;
> - the Secret stores the TLS certificate and private key.
>
> Keeping configuration and sensitive data outside the container image makes the application easier to configure and more secure.

You can also mount only specific keys by using the `items` field.

```yaml
volumes:
- name: config-volume
  configMap:
    name: nginx-config
    items:
    - key: nginx.conf
      path: nginx.conf
- name: secret-volume
  secret:
    secretName: nginx-tls
    items:
    - key: tls.crt
      path: /etc/nginx/tls/tls.crt
    - key: tls.key
      path: /etc/nginx/tls/tls.key
```

> [!CAUTION]
>
> A Secret or ConfigMap mounted as a Volume is not stored inside the container image.  
> Kubernetes creates the files dynamically and mounts them into the container's filesystem.
>
> If the Secret or ConfigMap is updated, the mounted files are automatically updated as well (with a small delay), without restarting the Pod, except when using `subPath` (and other particular cases).  
> Instead, when a Secret or ConfigMap is exposed as environment variable, the Pod must be recreated to see the new values.

```text
                ConfigMap / Secret
                        │
        ┌───────────────┴───────────────┐
        │                               │
        ▼                               ▼
Environment Variables             Mounted Volume
        │                               │
        ▼                               ▼
    env / envFrom                volume + volumeMount
        │                               │
        ▼                               ▼
Available at startup           Available as files
        │                               │
        ▼                               ▼
Pod restart required           Files automatically updated
(after changes)                (except subPath)
```

> [!TIP]
>
> Use a normal ConfigMap Volume when you want configuration updates to be propagated automatically to a running Pod.  
> Use `subPath` when you need to mount a single file to a specific location in the container's filesystem, but remember that updates to the ConfigMap will **not** be reflected until the Pod is recreated.

## Security Context

A security context defines privilege and access control settings for a Pod or Container. The most common are:

- `runAsUser`: specifies the Linux User ID (UID) used to run the container processes;
- `runAsGroup`: specifies the Linux Primary Group ID (GID) used to run the container processes;
- `runAsNonRoot`: prevents the container from running as the root user (UID 0);
- `fsGroup`: defines the GID assigned to mounted Volumes;
- Linux Capabilities: for example `NET_ADMIN`, `MAC_ADMIN` or `ALL`, which you can add or remove with `capabilities.add` and `capabilities.drop`.

Some of the SecurityContext parameters are definable at container level, some at pod level, while others are definable both at container and pod level.  
The following table shows where some of the main SecurityContext parameters are definable.

| Setting                    | Pod                | Container                               |
| -------------------------- | ------------------ | --------------------------------------: |
| `runAsUser`                | :white_check_mark: | :white_check_mark:                      |
| `runAsGroup`               | :white_check_mark: | :white_check_mark:                      |
| `runAsNonRoot`             | :white_check_mark: | :white_check_mark:                      |
| `fsGroup`                  | :white_check_mark: | :x:                                     |
| `supplementalGroups`       | :white_check_mark: | :x:                                     |
| `seLinuxOptions`           | :white_check_mark: | :white_check_mark:                      |
| `seccompProfile`           | :white_check_mark: | :white_check_mark:                      |
| `appArmorProfile`          | :white_check_mark: | :white_check_mark: (in modern versions) |
| `capabilities`             | :x:                | :white_check_mark:                      |
| `privileged`               | :x:                | :white_check_mark:                      |
| `allowPrivilegeEscalation` | :x:                | :white_check_mark:                      |
| `readOnlyRootFilesystem`   | :x:                | :white_check_mark:                      |

> [!NOTE]
>
> A Pod-level securityContext defines the default security settings for all containers in the Pod.
>
> A Container-level securityContext applies only to that container and overrides the Pod-level setting whenever the same field can be specified at both levels.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: web-pod
spec:
  securityContext:
    runAsUser: 1000
    runAsGroup: 3000
    runAsNonRoot: true
  containers:
  - name: ubuntu
    image: ubuntu
    command: ["sleep", "3600"]
    securityContext:
      capabilities:
        add:
        - NET_ADMIN
```

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: web-pod
spec:
  containers:
  - name: ubuntu
    image: ubuntu
    command: ["sleep", "3600"]
    securityContext:
      runAsUser: 1000
      runAsGroup: 3000
      runAsNonRoot: true
      capabilities:
        drop: ["ALL"]
```

If you want to check the user running a process, you can:

- list the processes with the `ps aux` command from the host machine;
- print the current user ID, group ID and belonging groups `kubectl exec [pod name] -- id`.
- print the current username `kubectl exec [pod name] -- whoami`.

## Resource requirements

When scheduling a Pod, the kube-scheduler considers the resource requests of the Pod and the available resources on each node.  
If a node does not have enough available resources, the kube-scheduler does not schedule the Pod onto that node and search for nodes with sufficient resources available.  
If no node has sufficient available resources, the Pod remains in the `Pending` state until suitable resources become available.

We can specify the amount of CPU and memory guaranteed on a container at creation with `resources.requests`.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: simple-webapp-color
spec:
  containers:
  - name: simple-webapp-color
    image: simple-webapp-color
    ports:
    - containerPort: 8080
    resources:
      requests:
        memory: "4Gi"
        cpu: 2
```

For CPU resources, you can specify values as low as `1m` (one millicore), which corresponds to `0.001` CPU.  
For example, the following values are equivalent:

- `0.1`
- `"0.1"`
- `"100m"`

Similarly, memory can be specified in different units.

For example, the following values represent approximately the same amount of memory:

- `256Mi`
- `268435456` (bytes)
- `268M`

You can also use:

- `Ki` and `K` (Kibibytes and Kilobytes)
- `Mi` and `M` (Mebibytes and Megabytes)
- `Gi` and `G` (Gibibytes and Gigabytes)
- `Ti` and `T` (Tebibytes and Terabytes)

> [!NOTE]
>
> Decimal units (`K`, `M`, `G`, `T`, `P`, `E`) are based on powers of **10**.
>
> Binary units (`Ki`, `Mi`, `Gi`, `Ti`, `Pi`, `Ei`) are based on powers of **2**.
>
> Kubernetes accepts both notations.

You can set a limit to the resources used by a container with `resources.limits`.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: simple-webapp-color
spec:
  containers:
  - name: simple-webapp-color
    image: simple-webapp-color
    ports:
    - containerPort: 8080
    resources:
      requests:
        memory: "1Gi"
        cpu: 2
      limits:
        memory: "2Gi"
        cpu: 2
```

If the limit is exceeded:

- CPU: the container is throttled (it simply cannot use more CPU time);
- Memory: it is typically terminated with an Out Of Memory (OOMKilled) event.

> [!CAUTION]
>
> By default a container has no limits or requests on resources;  
> this means that any container without limits set, can consume as many resources as it requires on any node and suffocate other pods or processes running on the node.

If limits are specified but requests are not, Kubernetes automatically sets the requests to the same values as the limits.  
Both `resources.requests` and `resources.limits` can be defined at container level (`spec.containers[*].resources`), not at pod level.

> [!NOTE]
>
> The scheduler considers only **resource requests**, not resource limits, when selecting a node for a Pod:
>
> - requests are used for **scheduling**;
> - limits are used for **enforcement** at runtime.
>
> ```text
>                Pod
>                 │
>         resources.requests
>                 │
>                 ▼
>   kube-scheduler chooses a node
>                 │
>                 ▼
>         Pod starts running
>                 │
>         resources.limits
>                 │
>                 ▼
>      kubelet / container runtime
>       enforce CPU and memory limits
> ```

You can also create objects called `LimitRange` at namespace level in order to define default resource requests and limits set for containers (and also PersistentVolumeClaim) when they are not specified inside the manifest.

```yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: resource-constraint
spec:
  limits:
  - default:
      cpu: 500m
      memory: 1Gi
    defaultRequest:
      cpu: 250m
      memory: 512Mi
    max:
      cpu: "1"
      memory: 1Gi
    min:
      cpu: 100m
      memory: 500Mi
    type: Container
```

Note that these limits are enforced when a pod is created.  
So if we create or change a LimitRange, it does not affect existing pods but only newly created pods.

> [!TIP]
>
> Not all fields (`default`, `defaultRequest`, `max`, `min`, `maxLimitRequestRatio`, etc.) are meaningful for every `type`.
>
> For example:
>
> - `default` and `defaultRequest` are intended for `type: Container`, as they define the default resource requests and limits applied to individual containers.  
> - For `type: Pod`, `min` and `max` are typically used instead. `default` and `defaultRequest` are not meaningful because a Pod does not define its own `resources` field.

Finally, you can also create ResourceQuota objects at namespace level to set hard limits for requests and limits.

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: my-resource-quota
spec:
  hard:
    requests.cpu: 4
    requests.memory: 4Gi
    limits.cpu: 10
    limits.memory: 10Gi
```

## Service Accounts

A service account is a type of non-human account that, in Kubernetes, provides a distinct identity in a Kubernetes cluster.

The concept of Service Account is linked to other security-related concepts in Kubernetes, such as authentication, authorization, RBAC, etc.  
In Kubernetes there are two types of accounts:

- user account: used by humans (an admin that performs administrative tasks or a developer deploying applications);
- service account: used by an application to interact with the Kubernetes cluster (a monitoring application like Prometheus or a build tool like Jenkins).

> [!IMPORTANT]
>
> A ServiceAccount **does not grant permissions by itself**.  
> Permissions are granted by binding the ServiceAccount to a Role or ClusterRole using RBAC.
>
> Pods authenticate to the Kubernetes API using a ServiceAccount.  
> What the Pod is allowed to do depends on the RBAC permissions assigned to that ServiceAccount, not on the ServiceAccount itself.

Some commands:

- `kubectl create serviceaccount dashboard-sa`: creates a ServiceAccount inside the default namespace;
- `kubectl get serviceaccount`: list the ServiceAccounts inside the default namespace;
- `kubectl describe serviceaccount dashboard-sa`: gives details of a ServiceAccount inside the default namespace.

### The default ServiceAccount

A service account named `default` already exists by default in every Kubernetes namespace.  
Whenever a Pod is created, the default service account is automatically mounted into the Pod as a projected volume on the path `/var/run/secrets/kubernetes.io/serviceaccount`.  
You can see that with the command `kubectl describe pod [pod name]`.

```bash
    Mounts:
      /var/run/secrets/kubernetes.io/serviceaccount from kube-api-access-srhgg (ro)
```

If you list the contents of that directory, you will see three files projected into the Pod

```console
$ kubectl exec -it nginx -- ls /var/run/secrets/kubernetes.io/serviceaccount
ca.crt  namespace  token
```

The default `ServiceAccount` is automatically mounted unless explicitly disabled with the setting `automountServiceAccountToken: false`.

> [!NOTE]
>
> By default, the `default` ServiceAccount has very limited permissions.  
> What it can do depends on the RBAC roles bound to it.

### Using a custom ServiceAccount

If you want to use a different `ServiceAccount`, specify its name into the `serviceAccountName` field.

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: dashboard-sa
  namespace: default
```

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-kubernetes-dashboard
spec:
  containers:
  - name: my-kubernetes-dashboard
    image: nginx
  serviceAccountName: dashboard-sa
```

```bash
kubectl create -f dashboard-sa.yaml
kubectl create -f my-kubernetes-dashboard.yaml
```

When you look at the pod details now, you see that the new `ServiceAccount` is being used

```console
$ kubectl describe pod my-kubernetes-dashboard
    Mounts:
      /var/run/secrets/kubernetes.io/serviceaccount from kube-api-access-bxmxs (ro)

$ kubectl exec -ti my-kubernetes-dashboard -- ls /var/run/secrets/kubernetes.io/serviceaccount
ca.crt  namespace  token
```

> [!IMPORTANT]
>
> A Pod can reference a ServiceAccount only at creation time.
>
> To change the ServiceAccount, recreate the Pod or update the Deployment so that a new rollout is triggered.

### ServiceAccount token

A ServiceAccount is mainly used to authenticate to the Kubernetes API.  
The kubelet automatically obtains the token and projects it into the Pod.

Inside the mounted `ServiceAccount` volume, the `token` file contains the JWT.

```console
$ kubectl exec -it nginx -- cat /var/run/secrets/kubernetes.io/serviceaccount/token
eyJhbGciOiJSUzI1NiIsImtpZCI6ImRSR3pRcTRSamFxTGdqS0hSZHhmN2w4RmtTMTFtZzdhUDJWSlB1NjNrVTgifQ.eyJhdWQiOlsiaHR0cHM6Ly9rdWJlcm5ldGVzLmRlZmF1bHQuc3ZjLmNsdXN0ZXIubG9jYWwiXSwiZXhwIjoxODE1MTQ1OTcyLCJpYXQiOjE3ODM2MDk5NzIsImlzcyI6Imh0dHBzOi8va3ViZXJuZXRlcy5kZWZhdWx0LnN2Yy5jbHVzdGVyLmxvY2FsIiwianRpIjoiYmRhOTNhNDItYjc5ZC00ZDU4LWFiMmQtMGZmYTdmN2I0ZThiIiwia3ViZXJuZXRlcy5pbyI6eyJuYW1lc3BhY2UiOiJkZWZhdWx0Iiwibm9kZSI6eyJuYW1lIjoibWluaWt1YmUiLCJ1aWQiOiIyNmEyOTM0MC0yN2VmLTQ0YzYtYjU0My04YmFlYWVjN2Y0Y2IifSwicG9kIjp7Im5hbWUiOiJuZ2lueCIsInVpZCI6IjVjNzExZDNhLTgxYzMtNDRhZS1hMWUxLTk5OGY1MzM0NmE5ZiJ9LCJzZXJ2aWNlYWNjb3VudCI6eyJuYW1lIjoiZGVmYXVsdCIsInVpZCI6IjE3ZWY0MDhiLTc4YTEtNGU1NC1iMDY0LWEzYzc1NTgyNjA1MiJ9LCJ3YXJuYWZ0ZXIiOjE3ODM2MTM1Nzl9LCJuYmYiOjE3ODM2MDk5NzIsInN1YiI6InN5c3RlbTpzZXJ2aWNlYWNjb3VudDpkZWZhdWx0OmRlZmF1bHQifQ.x_hvNFm86YWoZ29xXFinGhz7tzzZ83IflzQLZdu0TZMMoorFt9IgJGzMEcFnkuDecGYem8gSE-FAmhGss5JUq_5EE7dSrBsc-VF8sqYbGkBq6sSJDxmWkIC22qGcbGC0GcvdP0fffl98BZYs7Twc8B6KoGuO0KqcuwZH4bYsM31T19Gv1ubismH4-MqxM3d_mMhSCBQht_iymi8Bo6zLvGEPhHRgZzTH0u6nRF9BRTFsbeA7WLCcbuo-yAikEGjzbxS5w0LAEkROHc_Piv8ukfUMLrOf0V_66rOXDmE6EQdOHXna8wq37U2NuqEBO_AzUQnXrgJbw7NfcRK7iDweHg
```

You can use the JWT as a Bearer token when authenticating to the Kubernetes REST API (for example with CURL you can use it as authentication header).

If decoded, the JWT has the following structure

```json
{
  "aud": [
    "https://kubernetes.default.svc.cluster.local"
  ],
  "exp": 1815145972,
  "iat": 1783609972,
  "iss": "https://kubernetes.default.svc.cluster.local",
  "jti": "bda93a42-b79d-4d58-ab2d-0ffa7f7b4e8b",
  "kubernetes.io": {
    "namespace": "default",
    "node": {
      "name": "minikube",
      "uid": "26a29340-27ef-44c6-b543-8baeaec7f4cb"
    },
    "pod": {
      "name": "nginx",
      "uid": "5c711d3a-81c3-44ae-a1e1-998f53346a9f"
    },
    "serviceaccount": {
      "name": "default",
      "uid": "17ef408b-78a1-4e54-b064-a3c755826052"
    },
    "warnafter": 1783613579
  },
  "nbf": 1783609972,
  "sub": "system:serviceaccount:default:default"
}
```

we can notice an `exp` field which, unless specified otherwise, is one hour from the creation time.

You can also generate a new JWT with `kubectl create token [ServiceAccount name]`.

If a long-lived token is still required (similar to the legacy behavior, see later), we can still do that via a `Secret` of type `kubernetes.io/service-account-token` and the name of the `ServiceAccount` specified in the `metadata.annotations` section.

```yaml
apiVersion: v1
kind: Secret
type: kubernetes.io/service-account-token
metadata:
  name: mysecretname
  annotations:
    kubernetes.io/service-account.name: dashboard-sa
```

> [!CAUTION]
>
> Kubernetes says that a service-account-token Secret object should only be created if
>
> - we cannot use the `TokenRequest` API to obtain a token and
> - the security exposure of persisting non-expiring token credential is acceptable.

### A preview about RBAC, Role, RoleBinding

If you try to check if the token has the right to query the Kubernetes API with the command

```bash
kubectl auth can-i get pods --as=system:serviceaccount:default:dashboard-sa
```

you will receive a `no` as a response, because no permissions has assigned to it.

If, instead, you create a Role and assign it with a RoleBinding

```yaml
---
kind: Role
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  namespace: default
  name: pod-reader
rules:
- apiGroups:
  - ''
  resources:
  - pods
  verbs:
  - get
  - watch
  - list
```

```yaml
---
kind: RoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: read-pods
  namespace: default
subjects:
- kind: ServiceAccount
  name: dashboard-sa # Name is case sensitive
  namespace: default
roleRef:
  kind: Role #this must be Role or ClusterRole
  name: pod-reader # this must match the name of the Role or ClusterRole you wish to bind to
  apiGroup: rbac.authorization.k8s.io
```

```bash
kubectl apply -f pod-reader-role.yaml
kubectl apply -f dashboard-sa-role-binding.yaml
```

and re-execute the command above, you can see that now the response is `yes` because the token has sufficient rights.

```text
Application
      │
      ▼
ServiceAccount
      │
      ▼
JWT Token
      │
      ▼
Authentication
      │
      ▼
RBAC
      │
      ▼
Allowed / Denied
```

This topic will be covered covered in more detail later.

### Manually mount a ServiceAccount token

You manually mount a ServiceAccount token by disabling automatic mounting of API credentials and using a projected volume.

TODO: Complete this section.
      Check the last part of the lab on ServiceAccount.

### Evolution of ServiceAccount tokens

Service account tokens have evolved over time.

Prior to Kubernetes v1.24, creating a ServiceAccount automatically created a Secret containing a long-lived token.  
The Secret name appeared in the `tokens` field of the ServiceAccount.  
In that case, you can directly mount the Secret (containing the token) as a volume of a Pod (instead of exporting it and then injecting it in some way).

Until Kubernetes ≤ v1.21, that default token was not time bound (so it did not have an expiration and it is valid as long as the Service Account exists) and was not audience bound.  
In version 1.22 the TokenRequest API was introduced as part of the enhancement proposal KEP-1205, which introduced a more secure and scalable mechanism for provisioning ServiceAccount tokens.  
Tokens generated through this API are:

- time bound
- audience bound
- object bound

This makes `ServiceAccountTokens` significantly more secure.  
Instead of mounting a `Secret` containing a long-lived token, Kubernetes now obtains a short-lived token through the TokenRequest API and projects it directly into the Pod.

Kubernetes v1.24 introduced KEP-2799: "Reduction of Secret-based Service Account Tokens".  
Here, when creating a ServiceAccount, the Secret is not automatically created.  
You must run the command `kubectl create token dashboard-sa` to generate a token for that ServiceAccount, which will also be printed on screen at creation.

| Version | Behaviour                                                  |
| ------- | ---------------------------------------------------------- |
| ≤1.21   | Secret containing a long-lived token automatically created |
| 1.22    | TokenRequest API introduced                                |
| ≥1.24   | Secret no longer automatically created                     |

## Taints and Tolerations

Taints and Tolerations define Pod to node relationship.

- Taints are set on nodes and act as a repellent to Pods;
- Tolerations are set on pods and define the toleration on the taints.

By default, Pods have no tolerations and this means that Pods cannot be scheduled onto tainted nodes unless they define a matching toleration.  
In other words, a pod with no toleration will never be run on a node with a taint.

To define a taint on a node, perform the following command

```bash
kubectl taint nodes node-name [key]=[value]:[taint-effect]
```

The taint effects are:

- `NoSchedule`: the pod will not be scheduled on the node;
- `PreferNoSchedule`: the system will try to avoid placing a pod on a nod (but is not guaranteed);
- `NoExecute`: new pods will not be scheduled on the node and existing pods on the node if any will be evicted if they do not tolerate the taint.

For example

```bash
kubectl taint nodes node1 key1=value1:NoSchedule
```

places a taint on node `node1`. The taint has key `key1`, value `value1`, and taint effect `NoSchedule`. This means that no pod will be able to schedule onto `node1` unless it has a matching toleration.

> [!TIP]
>
> Control plane nodes, also if they have the capability to run pods, are not taken into account by the scheduler because, when the cluster is first set-up, a taint is automatically placed on them.
>
> ```console
> $ kubectl describe node controlplane | grep Taint
> Taints:             node-role.kubernetes.io/control-plane:NoSchedule
> ```
>
> This behavior can be modified as required but a best practice is to to not deploy applications on the master nodes.

You can then specify a toleration for a pod in the PodSpec.  
Both of the following tolerations "match" the taint created by the `kubectl taint` line above, and thus a pod with either toleration would be able to schedule onto `node1`:

```yaml
tolerations:
- key: "key1"
  operator: "Equal"
  value: "value1"
  effect: "NoSchedule"
```

```yaml
tolerations:
- key: "key1"
  operator: "Exists"
  effect: "NoSchedule"
```

To remove the taint added by the command above, you can run:

```bash
kubectl taint nodes node1 key1=value1:NoSchedule-
```

The default Kubernetes scheduler takes taints and tolerations into account when selecting a node to run a particular Pod.  
However, if you manually specify the `.spec.nodeName` for a Pod, that action bypasses the scheduler; the Pod is then bound onto the node where you assigned it, even if there are `NoSchedule` taints on that node that you selected.  
If this happens and the node also has a `NoExecute` taint set, the kubelet will eject the Pod unless there is an appropriate tolerance set.

> [!IMPORTANT]
>
> A toleration does **not** instruct Kubernetes to schedule a Pod onto a tainted node.  
> It only allows the Pod to be scheduled there.  
> The scheduler may still choose another node.

Here's an example of a pod that has some tolerations defined:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx
  labels:
    env: test
spec:
  containers:
  - name: nginx
    image: nginx
    imagePullPolicy: IfNotPresent
  tolerations:
  - key: "example-key"
    operator: "Exists"
    effect: "NoSchedule"
```

The default value for `operator` is `Equal`.

A toleration "matches" a taint if the keys are the same and the effects are the same, and:

- the `operator` is `Exists` (in which case no value should be specified), or
- the `operator` is `Equal` and the values should be equal.

> [!NOTE]
> There are two special cases:
>
> If `the` key is empty, then the `operator` must be `Exists`, which matches all keys and values. Note that the `effect` still needs to be matched at the same time.
>
> An empty `effect` matches all effects with key `key1`.

## Node Selectors and Node Affinity

To assign a Pod to a particular node using Node Selectors, we need to:

- label the node with `kubectl label nodes <node-name> <label-key>=<label-value>`;
- specify selectors inside the section `spec.nodeSelector` of the pod.

```bash
kubectl label nodes node-1 size=Large
```

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx
spec:
  containers:
  - name: nginx
    image: nginx
  nodeSelector:
    size: Large
    zone: EU
```

> [!TIP]
>
> NodeSelector supports only exact label matching (logical AND).  
> More complex expressions, like:
>
> - place the pod on a large or medium node;
> - place the pod on any node that is not small;
>
> require Node Affinity.

The Node Affinity functionality offers advanced features to ensure that pods are hosted on particular nodes.

The following example does the same as the previous one specified with `nodeSelector`

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx
spec:
  containers:
  - name: nginx
    image: nginx
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
        - matchExpressions:
          - key: size
            operator: In
            values:
            - Large
```

There are currently two types of node affinity:

- `requiredDuringSchedulingIgnoredDuringExecution`: the scheduler can't schedule the Pod unless the rule is met. This functions is like `nodeSelector`, but with a more expressive syntax;
- `preferredDuringSchedulingIgnoredDuringExecution`: the scheduler tries to find a node that meets the rule. If a matching node is not available, the scheduler still schedules the Pod.

> [!NOTE]
>
> In the preceding types, `IgnoredDuringExecution` means that if the node labels change after Kubernetes schedules the Pod, the Pod continues to run.

Inside `nodeSelectorTerms[*].matchExpressions` you define the key-value pairs in the form key - operator - value.

The types of operators are:

- `In`: The label value is present in the supplied set of strings
- `NotIn`: The label value is not contained in the supplied set of strings
- `Exists`: A label with this key exists on the object
- `DoesNotExist`: No label with this key exists on the object

If you want to allow the pod to be placed also on a node labeled with medium, you can just add it to the Values

```yaml
        - matchExpressions:
          - key: size
            operator: In
            values:
            - Large
            - Medium
```

If, instead, you want to avoid the pod to be placed on a node labeled with small, you can use the following syntax

```yaml
        - matchExpressions:
          - key: size
            operator: NotIn
            values:
            - Small
```

And now for an example in which to use the `exists` operator: Medium and Large nodes are labeled while small nodes aren't.  
The following example gives the same result.

```yaml
        - matchExpressions:
          - key: size
            operator: Exists
```

We don't need the value set in this snippet as the `Exists` operator does not compare the values.

| Feature              | nodeSelector       | nodeAffinity       |
| -------------------- | ------------------ | ------------------ |
| Equality match       | :white_check_mark: | :white_check_mark: |
| OR conditions        | :x:                | :white_check_mark: |
| NOT conditions       | :x:                | :white_check_mark: |
| Preferred scheduling | :x:                | :white_check_mark: |

## Multi-Container Pods and InitContainers

A Pod can contain more than one container.  
All containers in the same Pod:

- share the same network namespace;
- share the same IP address;
- can share volumes;
- have the same lifecycle.

There are different patterns of multi-container pods:

- co-located containers: the services are dependent on each other and share their entire lifecycle inside the pod;
- init containers: there are initialization steps to be performed when the pod starts (for example waiting for a DB to be ready or pulling a code or binary from a repository that will be used by the main web application);
- sidecar container: the sidecar starts first, does its job and continue to live along the main application container and ends after the main apps ends (for example a log shipper).

In the co-located containers, there is no guarantee that a container starts before another.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: simple-webabb
  labels:
    name: simple-webabb
spec:
  containers:
  - name: web-app
    image: web-app
    ports:
    - containerPort: 8080
  - name: main-app
    image: main-app
```

With the `InitContainer` you can also define more than one init containers; in that case each init container is run one at a time in sequential order.  
In the following example, first runs the db-checker and then ends; then runs the api-checker and ends; finally runs the main application.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: simple-webabb
  labels:
    name: simple-webabb
spec:
  containers:
  - name: web-app
    image: web-app
    ports:
    - containerPort: 8080
  initContainers:
  - name: db-checker
    image: busybox
    command: ["sh","-c","wait-for-db-to-start.sh"]
  - name: api-checker
    image: busybox
    command: ["sh","-c","wait-for-another-api.sh"]
```

If any of the initContainers fail to complete, Kubernetes restarts the Pod repeatedly until the Init Container succeeds.

In order to achieve the sidecar container pattern, you need to add the attribute `restartPolicy: Always` that also ensures the `initContainer` will continue to run and will be terminated only after the main application stops (in this way the log shipper can catch the startup and termination logs of the main container).

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: simple-webabb
  labels:
    name: simple-webabb
spec:
  containers:
  - name: simple-app
    image: simple-app
    ports:
    - containerPort: 8080
    volumeMounts:
    - mountPath: /log
      name: log-volume
  initContainers:
  - name: log-shipper
    image: filebeat
    volumeMounts:
    - name: log-volume
      mountPath: /var/log/event-simulator/
    restartPolicy: Always
```

## Observability

### Pod Status, Pod Phases and Pod Conditions

Every Pod has a `status` field, which is a `PodStatus` object, which contains information about the current state of the Pod, including:

- the Pod phase;
- the Pod conditions;
- container statuses;
- IP addresses;
- start time;
- and other runtime information.

You can get details about the Pod Status with the command `kubectl describe pod <pod name>`.

```text
Pod
 │
 └── status (PodStatus)
      │
      ├── phase
      │     ├── Pending
      │     ├── Running
      │     ├── Succeeded
      │     ├── Failed
      │     └── Unknown
      │
      ├── conditions
      │     ├── PodScheduled
      │     ├── PodReadyToStartContainers
      │     ├── Initialized
      │     ├── ContainersReady
      │     └── Ready
      │
      ├── podIP
      ├── startTime
      ├── hostIP
      └── containerStatuses
```

The phase of a Pod is a simple, high-level summary of where the Pod is in its lifecycle:

- `Pending`: the Pod has been accepted by the Kubernetes cluster, but one or more of the containers has not been set up and made ready to run. This includes the time a Pod spends waiting to be scheduled as well as the time spent downloading container images over the network;
- `Running`: the Pod has been bound to a node, and all of the containers have been created. At least one container is still running, or is in the process of starting or restarting;
- `Succeeded`: all containers in the Pod have terminated in success, and will not be restarted;
- `Failed`: all containers in the Pod have terminated, and at least one container has terminated in failure. That is, the container either exited with non-zero status or was terminated by the system, and is not set for automatic restarting;
- `Unknown`: for some reason the state of the Pod could not be obtained. This usually indicates a communication problem between the control plane and the node where the Pod should be running.

> [!TIP]
>
> The `STATUS` column displayed by `kubectl get pods` does not always correspond to the Pod phase and its instead an "user-friendly" representation built by kubectl.
>
> For example, sometimes you receive `ContainerCreating`, which is a **Waiting Reason**, not a Pod phase.  
> It typically appears after the Pod has been scheduled while Kubernetes is pulling container images, creating the Pod sandbox, configuring networking and starting the containers.  
> During this time, the Pod phase is typically still `Pending`.
>
> ```console
> $ kubectl get pod nginx
>
> NAME    READY   STATUS               RESTARTS   AGE
> nginx   0/1     ContainerCreating    0          3s
> ```
>
> ```console
> $ kubectl describe pod nginx
>
> Status:    Pending
> ...
> Container State:
>   Waiting
>     Reason: ContainerCreating
> ```
>
> | Pod Phase | STATUS showed by `kubectl get pods` |
> | --------- | ----------------------------------- |
> | Pending   | Pending                             |
> | Pending   | ContainerCreating                   |
> | Pending   | ErrImagePull                        |
> | Pending   | ImagePullBackOff                    |
> | Running   | Running                             |
> | Running   | CrashLoopBackOff                    |
> | Running   | Terminating                         |

Pod's status includes an array of `PodConditions` (of true or false values) that indicate whether the Pod has passed certain checkpoints.  
As a Pod progresses through its lifecycle, the Kubelet sets the following conditions roughly in this order:

- `PodScheduled`: the pod has been scheduled on a node;
- `PodReadyToStartContainers`: the Pod sandbox has been successfully created and networking configured. The sandbox and network are set up by the container runtime and CNI plugin;
- `Initialized`: all `init containers` have completed successfully (for a Pod without init containers, this is set to True before sandbox creation);
- `ContainersReady`: all containers in the pod are ready;
- `Ready`: the Pod is able to serve requests and should be added to the load balancing pools of all matching Services.

Not every Pod will necessarily expose all conditions, depending on the Kubernetes version and enabled features.

When running `kubectl describe pod <pod name>`, the Pod Conditions appears like this:

```bash
Conditions:
  Type                        Status
  PodReadyToStartContainers   True
  Initialized                 True
  Ready                       True
  ContainersReady             True
  PodScheduled                True
```

Here is a sequence a Pod should incur in order to pass from the Pending Pod Phase to the Running one, through the different Pod Conditions.

```text
Pod created
      │
      ▼
 Pending
      │
      ▼
 PodScheduled = True
      │
      ▼
 PodReadyToStartContainers = True
      │
      ▼
 Initialized = True
      │
      ▼
 ContainersReady = True
      │
      ▼
 Ready = True
      │
      ▼
 Running
```

By default, Kubernetes assumes that as soon as the container is created, it is ready to serve requests.  
However, many applications need additional time to initialize before they are actually ready to serve requests.  
Without a readiness probe, Kubernetes considers the container ready as soon as it starts, so Services may begin routing traffic to an application that is still initializing.  
Readiness probes solve this problem by tying the Pod's `Ready` condition to the actual health of the application.

### Readiness and Liveness Probes

As a developer of the application, you know better what it means for the application to be ready.

There are different ways to test / probe if an application inside a container is actually ready by adding a `readinessProbe` in the Pod manifest.  
Some of them are:

- testing a call to an API through http in case of a web application;

  ```yaml
      readinessProbe:
        httpGet:
          path: /api/ready
          port: 8080
  ```

- when a particular TCP Socket is listening, on a Database;

  ```yaml
      readinessProbe:
        tcpSocket:
          port: 3306
  ```

- executing a command in a custom script that exits successfully when the application is ready.

  ```yaml
      readinessProbe:
        exec:
          command:
          - sh
          - -c
          - test -f /app/is_ready
  ```

Until the `readinessProbe` is not completed positively, the Service does not forward any traffic to the Pod.

There are other options, like:

- add an additional delay to the probe (for example if you know that the application takes a minimum amount of time to warm-up);
- specify how often to probe with `periodSeconds` option;
- specify a different threshold of consecutive failures before Kubernetes considers the probe failed (if not specified is three).

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: simple-webabb
  labels:
    name: simple-webabb
spec:
  containers:
  - name: simple-app
    image: simple-app
    ports:
    - containerPort: 8080
    readinessProbe:
      httpGet:
        path: /api/ready
        port: 8080
      initialDelaySeconds: 10
      periodSeconds: 5
      failureThreshold: 8
```

Readiness Probes are particularly useful in a multi-pod setup.  
For example when you have a ReplicaSet with multiple pods, the Service routes traffic only to the ready pods.

A Liveness probe can be configured on a container to periodically test wether the application in the container is healthy.  
If the test fails, the container is considered unhealthy and it will be destroyed and recreated.

Again, as a developer of the application, you must define what it means for the application to be healthy.

The Liveness Probe is defined in the Pod definition file similarly as already did with the readiness probe.

```yaml
      livenessProbe:
        exec:
          command:
          - cat
          - /app/is_healthy.sh
```

Similarly to readiness probes, we have:

- `httpGet` option for API, `tcpSocket` for port and `exec` for commands;
- `initialDelaySeconds`, `periodSeconds` and `failureThreshold` options.

> [!TIP]
>
> | Probe type | Objective            |
> | ---------- | -------------------- |
> | Readiness  | Can receive traffic? |
> | Liveness   | Should be restarted? |

### Monitoring, Logging, and Debugging

Kubernetes provides several commands to inspect logs, monitor resource usage and troubleshoot applications.

For the logs we can use `kubectl logs`.  
By default, the logs are retrieved from the main container but we can select a specific container using the `--container` or `-c` option.  
We can also retrieve the logs from all the containers of a pod with the option `--all-containers`.

There are also options for retrieving useful metrics via the built-in `Metrics Server`.  
You can have one Metrics Server per cluster.  
The Metrics Server periodically retrieves resource usage metrics from the Kubelets running on each node, aggregates them and exposes them through the Kubernetes Metrics API.  
Each Kubelet embeds `cAdvisor`, which collects CPU and memory usage statistics from the running containers.

With Minikube we can enable it with the command `minikube addons enable metrics-server`.  
For the other environments, you can run `kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml`.

> [!NOTE]
>
> Minikube already provides the Metrics Server as an addon.  
> In production clusters it must be deployed separately.

Once this component is running, you can see useful information with the command `kubectl top node [node]` or `kubectl top pod [pod]`.

```console
$ kubectl top node
NAME           CPU(cores)   CPU(%)   MEMORY(bytes)   MEMORY(%)
controlplane   184m         1%       895Mi           1%
node01         24m          0%       168Mi           0%
```

```console
$ kubectl top pod
NAME       CPU(cores)   MEMORY(bytes)
elephant   13m          30Mi
lion       1m           16Mi
rabbit     104m         250Mi
```

> [!TIP]
>
> A common troubleshooting workflow is:
>
> 1. `kubectl get pods`
> 2. `kubectl describe pod <pod>`
> 3. `kubectl logs <pod>`
> 4. `kubectl exec -it <pod> -- sh`
>
> In many cases, these four commands are sufficient to identify the cause of a failing Pod.

Useful commands:

- `kubectl logs <pod>`: displays the logs produced by the main container of a Pod;
- `kubectl logs -f <pod>`: streams the logs in real time (similar to `tail -f`);
- `kubectl logs <pod> -c <container>`: displays the logs of a specific container in a multi-container Pod;
- `kubectl logs --previous <pod>`: displays the logs of the previously terminated container instance (useful for troubleshooting `CrashLoopBackOff`);
- `kubectl exec <pod> -- <command>`: executes a command inside a running container of a Pod;
- `kubectl describe <resource>`: displays detailed information about a resource, including its configuration, status, events and conditions;
- `kubectl top pod`: displays the current CPU and memory usage of Pods;
- `kubectl top node`: displays the current CPU and memory usage of Nodes.

`kubectl top` relies on the **Metrics Server**, a cluster-wide component that collects CPU and memory usage from the Kubelets running on each node.  
The Metrics Server periodically retrieves resource usage metrics and exposes them through the Kubernetes Metrics API.  
Without the Metrics Server installed, commands such as `kubectl top pod` and `kubectl top node` will fail because no resource metrics are available.

> [!NOTE]
>
> The Metrics Server provides **current resource usage** (CPU and memory) only.  
> It is **not** a full monitoring solution and does not store historical metrics because it keeps metrics only in memory and does not persist them to disk.
>
> For long-term monitoring and alerting, tools such as Prometheus and Grafana are typically used.

## Labels, Selectors and Annotations

Labels and selectors are the standard Kubernetes mechanism used to organize, identify and select objects.  
Labels are key/value pairs attached to Kubernetes objects.  
You can assign as many labels as you need to each object.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: simple-webapp
  labels:
    app: app1
    function: front-end
spec:
  containers:
  - name: simple-webapp
    image: simple-webapp
    ports:
    - containerPort: 80
```

In the example above, we added following label keys and values on a pod

| Key      | Value     |
| -------- | --------- |
| app      | app1      |
| function | front-end |

You can then use selectors to filter the output of kubectl commands based on labels.

```bash
kubectl get pods --selector app=App1
```

You can also filter using multiple labels by separating them with commas.  
The short form of `--selector` is `-l`.

```bash
kubectl get pod -l env=prod,bu=finance,tier=frontend
```

Kubernetes objects use labels and selectors internally to connect different objects together.  
For example the following command for creating a deployment

```bash
kubectl create deployment nginx --image=nginx --replicas=3
```

automatically creates:

- a label (`app: nginx`) on the Deployment;
- the same label on the Pod template;
- a selector (`spec.selector.matchLabels`) matching that label.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: nginx
  ...
spec:
  ...
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      ...
status:
  ...
```

The Deployment manages Pods by selecting them through the labels defined in the Pod template.

> [!TIP]
>
> Although a single label is often sufficient, using multiple labels generally makes selectors more precise and expressive.  
> For example, selecting only `app=web` may match frontend Pods, backend Pods, Jobs or CronJobs.  
> Combining multiple labels such as `app=web,tier=frontend,env=prod` reduces the chance of selecting unintended objects.
>
> You can also use `matchExpressions` instead of `matchLabels` to have more granular control over selectors.

When a Service is created, it uses its selector to match the labels of the target Pods.  
In many cases those Pods are created by a ReplicaSet or Deployment, but Services always select Pods directly.

> [!IMPORTANT]
>
> Services, ReplicaSets, Deployments and NetworkPolicies use **labels and selectors** to identify Pods.

Annotations are arbitrary key/value pairs used to attach non-identifying metadata to Kubernetes objects.  
Unlike labels, annotations are not used for selection or grouping.  
Typical examples include:

- build information;
- Git commit SHA;
- owner or contact details;
- deployment timestamps;
- configuration for external tools such as Ingress controllers, Prometheus, ArgoCD, Helm, etc.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-web-pod
  namespace: production
  labels:
    app: web
    tier: frontend
  annotations:
    buildVersion: 1.34
    imageregistry: "https://docker.com"
    description: "Main frontend Nginx web server"
spec:
  containers:
  - name: nginx-container
    image: nginx:1.21.6
```

## Deployment Updates and Rollback

When you first create a Deployment, it triggers a rollout.  
A new rollout creates a new ReplicaSet, which is recorded as a new Deployment revision.  
Whenever the Pod template (`spec.template`) changes (for example the container image, environment variables, commands and arguments, volumes, resource limits, labels, annotations, etc), a new rollout is triggered and a new Deployment revision is created.  
This helps us keep track of the changes made to our deployment, enabling rollback capabilities if necessary.

> [!IMPORTANT]
>
> Only changes to the Pod template (`spec.template`) create a new ReplicaSet.  
> Scaling a Deployment (`spec.replicas`) updates the existing ReplicaSet but does **not** create a new ReplicaSet or a new Deployment revision.

You can see the status of a rollout with the command `kubectl rollout status deployment/myapp`.  
To see the rollout history and the available revisions, use `kubectl rollout history Deployment myapp`.  
To see the details of a specific revision use the command `kubectl rollout history deployment/myapp --revision=2`.

> [!NOTE]
>
> You can use interchangeably `deployment/deployment_name`, `Deployment deployment_name`, `deployment deployment_name`, `deploy deployment_name`

In Kubernetes, there are two types of deployment strategies:

- Recreate: scales the old ReplicaSet down to zero before scaling the new ReplicaSet up to the desired amount, causing downtime during the update;
- RollingUpdate: the old ReplicaSet is gradually scaled down while the new ReplicaSet is gradually scaled up, gradually replacing the old Pods with new Pods until all Pods have been replaced, ensuring that part of the application remains available during the update.

The Deployment strategy is configured under `spec.strategy` and `strategy.type` determines how Pods are replaced during a Deployment update; the default deployment strategy is the RollingUpdate.

```yaml
spec:
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 25%
      maxUnavailable: 25%
```

```yaml
spec:
  strategy:
    type: Recreate
```

> [!NOTE]
>
> The `rollingUpdate` field is used only when `type: RollingUpdate`.
> It is ignored when `type: Recreate`.

`maxSurge` defines how many extra Pods can be created above the desired number of replicas during a rolling update.  
For example, if Deployment replicas is set to 8 and `maxSurge: 25%`, Kubernetes may temporarily run 8 old Pods + 2 new Pods = 10 Pods.

`maxUnavailable` instead, defines how many Pods are allowed to be unavailable during the update.  
For example, if Deployment replicas is set to 8 and `maxUnavailable: 25%`, Kubernetes may temporarily have 6 available Pods and 2 unavailable Pods.

Both `maxSurge` `maxUnavailable` can be specified as an absolute number or a percentage and their default value is 25%.

In an example of RollingUpdate with the following specs

- Deployment replicas: 8
- maxSurge: 2
- maxUnavailable: 2

the steps for switching from the old ReplicaSet to the new one are the following

| Update Step | 1   | 2   | 3   | ... | Final |
| ----------- | --- | --- | --- | --- | ----- |
| Old Pods    | 8   | 6   | 6   | ... | 0     |
| New Pods    | 2   | 2   | 4   | ... | 8     |
| Total Pods  | 10  | 8   | 10  | ... | 8     |

In order to update a Deployment, you can modify its manifest and then apply the changes with `kubectl apply -f <manifest_file>`.  
You can also use the command `kubectl set image Deployment myapp nginx=nginx:1.27` (where `nginx` is the container name inside the Deployment) if you just need to change the image of a container in the Deployment.

Kubernetes Deployments allow us to rollback a Deployment to a previous revision, with the `kubectl rollout undo` command, for example:

- `kubectl rollout undo deployment/myapp`: if you want to rollback to the previous Deployment revision;
- `kubectl rollout undo Deployment myapp --to-revision=2`: if you want to rollback to a specific revision.

The Deployment then removes the Pods belonging to the current ReplicaSet and scales the previous ReplicaSet back up.

```text
kubectl create deployment
Deployment --► Revision 1 --► ReplicaSet #1 --► Pods v1

kubectl set image / kubectl apply / kubectl edit
Deployment --► Revision 2 --► ReplicaSet #2 --► Pods v2

kubectl rollout undo
ReplicaSet #1 becomes active again and ReplicaSet #2 is scaled down
```

Finally, if you need to restart the pods of the current revision of a deployment (like to force a Pod restart after an update of a ConfigMap or a Secret exposed through environment variables) you can use the command `kubectl rollout restart deployment/myapp`.  
`kubectl rollout restart` updates the Deployment by adding (or updating) the annotation `kubectl.kubernetes.io/restartedAt` in the Pod template (`spec.template.metadata.annotations`).  
Since the Pod template changes, even though the application configuration itself remains unchanged, Kubernetes creates a new ReplicaSet and performs a new rollout.

> [!TIP]
>
> - ReplicaSets should rarely be managed directly; they are normally managed by Deployments;
> - Deployments manage ReplicaSets and provide rolling updates and rollback capabilities;
> - only changes to spec.template create a new ReplicaSet;
> - scaling a Deployment does not create a new revision;
> - `kubectl rollout undo` rolls back the Deployment;
> - `kubectl rollout restart` forces a new rollout without changing the application.

Here are some handy examples related to updating a Kubernetes Deployment:

- **Creating a deployment, checking the rollout status, history and details**
  
  We will first create a simple deployment
  
  ```console
  $ kubectl create deployment nginx --image=nginx:1.30
  deployment.apps/nginx created
  ```
  
  Now inspecting the rollout status and the rollout history
  
  ```console
  $ kubectl rollout status deployment nginx
  Waiting for deployment "nginx" rollout to finish: 0 of 1 updated replicas are available...
  deployment "nginx" successfully rolled out
  
  $ kubectl rollout history deployment nginx
  deployment.apps/nginx
  REVISION  CHANGE-CAUSE
  1         <none>
  ```

  Finally, here we analyze the details of the deployment

  ```console
  $ kubectl describe deployment nginx
  Name:                   nginx
  Namespace:              default
  CreationTimestamp:      Fri, 24 Jul 2026 09:47:02 +0200
  Labels:                 app=nginx
  Annotations:            deployment.kubernetes.io/revision: 1
  Selector:               app=nginx
  Replicas:               1 desired | 1 updated | 1 total | 1 available | 0 unavailable
  StrategyType:           RollingUpdate
  MinReadySeconds:        0
  RollingUpdateStrategy:  25% max unavailable, 25% max surge
  Pod Template:
    Labels:  app=nginx
    Containers:
    nginx:
      Image:         nginx:1.30
      Port:          <none>
      Host Port:     <none>
      Environment:   <none>
      Mounts:        <none>
    Volumes:         <none>
    Node-Selectors:  <none>
    Tolerations:     <none>
  Conditions:
    Type           Status  Reason
    ----           ------  ------
    Available      True    MinimumReplicasAvailable
    Progressing    True    NewReplicaSetAvailable
  OldReplicaSets:  <none>
  NewReplicaSet:   nginx-7c5b6c9475 (1/1 replicas created)
  Events:
    Type    Reason             Age   From                   Message
    ----    ------             ----  ----                   -------
    Normal  ScalingReplicaSet  6s    deployment-controller  Scaled up replica set nginx-7c5b6c9475 from 0 to 1
  ```

  in the output you can notice interesting fields like `Replicas`, `StrategyType`, `RollingUpdateStrategy`, `OldReplicaSets` and `NewReplicaSets`.

- **Using the `--revision` flag**
  
  Here the revision 1 is the first version where the deployment was created.  
  You can check the status of each revision individually by using the `--revision` flag:
  
  ```console
  $ kubectl rollout history deployment nginx --revision=1
  deployment.apps/nginx with revision #1
  Pod Template:
    Labels:       app=nginx
          pod-template-hash=7c5b6c9475
    Containers:
    nginx:
      Image:      nginx:1.30
      Port:       <none>
      Host Port:  <none>
      Environment:        <none>
      Mounts:     <none>
    Volumes:      <none>
    Node-Selectors:       <none>
    Tolerations:  <none>
  
  ```

- **Using the `--record` flag**

  > [!NOTE]
  >
  > Older Kubernetes versions supported the `--record` flag to automatically populate the `kubernetes.io/change-cause` annotation.
  >
  > The flag has currently been deprecated.  
  > If desired, the annotation can still be added manually.
  
  You would have noticed that the "change-cause" field is empty in the rollout history output.  
  We can use the `--record` flag to save the command used to create / update a deployment against the revision number.
  
  ```console
  $ kubectl set image deployment nginx nginx=nginx:1.29 --record
  Flag --record has been deprecated, --record will be removed in the future
  deployment.apps/nginx image updated
  
  $ kubectl rollout history deployment nginx
  deployment.apps/nginx
  REVISION  CHANGE-CAUSE
  1         <none>
  2         kubectl set image deployment nginx nginx=nginx:1.29 --record=true
  ```
  
  You can now see that the change-cause is recorded for the revision 2 of this deployment.

  Let's make some more changes.  
  In the example below, we are editing the deployment and changing the image from `nginx:1.29` to `nginx:latest` while making use of the `--record` flag.
  
  ```console
  $ kubectl edit deployments nginx --record
  Flag --record has been deprecated, --record will be removed in the future
  deployment.apps/nginx edited
  
  $ kubectl rollout history deployment nginx
  deployment.apps/nginx
  REVISION  CHANGE-CAUSE
  1         <none>
  2         kubectl set image deployment nginx nginx=nginx:1.29 --record=true
  3         kubectl edit deployments nginx --record=true
  
  $ kubectl rollout history deployment nginx --revision=3
  deployment.apps/nginx with revision #3
  Pod Template:
    Labels:       app=nginx
          pod-template-hash=fffb5cbb8
    Annotations:  kubernetes.io/change-cause: kubectl edit deployments nginx --record=true
    Containers:
    nginx:
      Image:      nginx:latest
      Port:       <none>
      Host Port:  <none>
      Environment:        <none>
      Mounts:     <none>
    Volumes:      <none>
    Node-Selectors:       <none>
    Tolerations:  <none>
  ```

- **Undo a change**

  Lets now rollback to the previous revision:

  ```console
  $ kubectl rollout undo Deployment nginx
  deployment.apps/nginx rolled back
  
  $ kubectl rollout history deployment nginx
  deployment.apps/nginx
  REVISION  CHANGE-CAUSE
  1         <none>
  3         kubectl edit deployments nginx --record=true
  4         kubectl set image deployment nginx nginx=nginx:1.29 --record=true
  ```

  > [!IMPORTANT]
  >
  > Rolling back to a previous revision does **not** reactivate that revision.
  >
  > Kubernetes creates a **new Deployment revision** whose Pod template matches the selected historical revision.
  
  With this, we have rolled back to the previous version of the deployment with the image = nginx:1.29.

  ```console
  $ kubectl describe deployments nginx | grep -i image:
      Image:        nginx:1.29
  ```
  
  To rollback to specific revision we will use the `--to-revision` flag.  
  With `--to-revision=1`, it will be rolled back with the first image we used to create a deployment as we can see in the rollout history output.

  ```console
  $ kubectl rollout history deployment nginx --revision=1
  deployment.apps/nginx with revision #1
  Pod Template:
    Labels:       app=nginx
          pod-template-hash=7c5b6c9475
    Containers:
    nginx:
      Image:      nginx:1.30
      Port:       <none>
      Host Port:  <none>
      Environment:        <none>
      Mounts:     <none>
    Volumes:      <none>
    Node-Selectors:       <none>
    Tolerations:  <none>
  
  $ kubectl rollout undo deployment nginx --to-revision=1
  deployment.apps/nginx rolled back
  ```
  
  We can check the current image used in the deployment

  ```console
  $ kubectl describe deployments nginx | grep -i image:
  Image: nginx:1.30
  ```

  and the details of the deployment updated with the new ReplicaSets and Events

  ```console
  $ kubectl describe deployment nginx
  Name:                   nginx
  Namespace:              default
  CreationTimestamp:      Fri, 24 Jul 2026 10:16:43 +0200
  Labels:                 app=nginx
  Annotations:            deployment.kubernetes.io/revision: 5
  Selector:               app=nginx
  Replicas:               1 desired | 1 updated | 1 total | 1 available | 0 unavailable
  StrategyType:           RollingUpdate
  MinReadySeconds:        0
  RollingUpdateStrategy:  25% max unavailable, 25% max surge
  Pod Template:
    Labels:  app=nginx
    Containers:
    nginx:
      Image:         nginx:1.30
      Port:          <none>
      Host Port:     <none>
      Environment:   <none>
      Mounts:        <none>
    Volumes:         <none>
    Node-Selectors:  <none>
    Tolerations:     <none>
  Conditions:
    Type           Status  Reason
    ----           ------  ------
    Available      True    MinimumReplicasAvailable
    Progressing    True    NewReplicaSetAvailable
  OldReplicaSets:  nginx-6884758f6f (0/0 replicas created), nginx-fffb5cbb8 (0/0 replicas created)
  NewReplicaSet:   nginx-7c5b6c9475 (1/1 replicas created)
  Events:
    Type    Reason             Age                From                   Message
    ----    ------             ----               ----                   -------
    Normal  ScalingReplicaSet  69s                deployment-controller  Scaled down replica set nginx-7c5b6c9475 from 1 to 0
    Normal  ScalingReplicaSet  49s                deployment-controller  Scaled up replica set nginx-fffb5cbb8 from 0 to 1
    Normal  ScalingReplicaSet  35s (x2 over 70s)  deployment-controller  Scaled up replica set nginx-6884758f6f from 0 to 1
    Normal  ScalingReplicaSet  33s                deployment-controller  Scaled down replica set nginx-fffb5cbb8 from 1 to 0
    Normal  ScalingReplicaSet  10s (x2 over 89s)  deployment-controller  Scaled up replica set nginx-7c5b6c9475 from 0 to 1
    Normal  ScalingReplicaSet  9s (x2 over 47s)   deployment-controller  Scaled down replica set nginx-6884758f6f from 1 to 0
  ```

### Additional Deployment strategies

Besides the built-in Deployment strategies (`RollingUpdate` and `Recreate`), there are other application deployment strategies commonly used in production, such as:

- Blue Green deployment;
- Canary deployment.

These are not configurable in the Kubernetes Deployment manifest, but they can be implemented in other ways.  
They are best implemented with service meshes like Istio, but let's see ho to implement with Kubernetes primitives.

The **Blue Green deployment** procedure:

1) the old version (blue) is deployed;
2) all the traffic is routed to the old (blue) version;
3) the new version (green) is deployed alongside the old version (blue);
4) tests are performed on the new version while public traffic is still routed to the old version;
5) once all tests are passed, we switch traffic to the new (green) version, all at once.

and we can implement it in K8s in this way:

1) create the first deployment called `blue` with the label `version: v1`;

    ```yaml
    kind: Deployment
    metadata:
      name: blue
      labels:
        version: v1
    ```

2) create a service with the selector set as `version: v1`;

    ```yaml
    kind: Service
    spec:
      selector:
        version: v1
    ```
  
3) create the second deployment called `green` with the label `version: v2`;

   ```yaml
    kind: Deployment
    metadata:
      name: green
      labels:
        version: v2
   ```

4) perform validation tests against the Green deployment;
5) change the selector on the service as `version: v2`.

    ```yaml
    kind: Service
    spec:
      selector:
        version: v2
    ```

The **canary deployment** procedure is the following:

1) the old version is deployed at full desired replicas;
2) all the traffic is routed to the old version (primary);
3) the new version is deployed with few replicas (canary)
4) we start to route only a small portion of traffic to the canary deployment;
5) tests are performed on the new version, which is partly live with its few replicas;
6) once all tests are passed, we upgrade the primary deployment with the new version of the application (maybe making use of rollingUpdate strategy);
7) we dispose the canary deployment.

and we can implement it in K8s in this way:

1) create the first deployment called `primary` with a full amount of replicas (for example 5 pods) with the labels `version: v1` and `app:front-end`;

    ```yaml
    kind: Deployment
    metadata:
      name: primary
      labels:
        version: v1
        app: front-end
    spec:
      replicas: 5
    ```

2) create a service with the selector set as `version: v1`;

    ```yaml
    kind: Service
    spec:
      selector:
        version: v1
    ```

3) create the second deployment called `canary` with the label `version: v2` and a reduced amount of replicas (for example 1 pod) with the labels `version: v2` and `app:front-end`;

    ```yaml
    kind: Deployment
    metadata:
      name: canary
      labels:
        version: v2
        app: front-end
    spec:
      replicas: 1
    ```

4) change the Service selector to `app:front-end`, so that both Deployments become Endpoints of the same Service, with a smaller percentage on the canary one;

    ```yaml
    kind: Service
    spec:
      selector:
        app: front-end
    ```

5) tests are performed on the new version;
6) once we are confident, we upgrade the pod version of the primary deployment to the new one;

    ```bash
    kubectl set image deployment primary container-name=image:2.0
    ```

7) the canary deployment is deleted.

    ```bash
    kubectl delete deployment canary
    ```

One of the caveat of the canary deployment is that we have limited control over the split of traffic through the deployments.  
The traffic split is going to be governed by the number of pods present in each deployment.  
For example, we cannot route 1% of traffic to the canary deployment (we could if we have 100 pods).  
A service mesh such as Istio allows routing traffic based on explicit percentages (for example 1%, 5%, 10%, 25%, 50%, etc.), independently of the number of Pods backing each Deployment.

> [!IMPORTANT]
>
> Kubernetes Services do **not** support weighted traffic routing.  
> When multiple Pods match the Service selector, requests are distributed among the available endpoints.  
> The traffic distribution therefore depends mainly on the number of Pods backing each Deployment.

| Pattern       | Downtime | Traffic shift    | Rollback                   |
| ------------- | -------- | ---------------- | -------------------------- |
| Recreate      | Yes      | Immediate        | Slow                       |
| RollingUpdate | No       | Progressive      | Automatic (`rollout undo`) |
| Blue-Green    | No       | Immediate switch | Very fast                  |
| Canary        | No       | Progressive      | Very fast                  |

## Jobs and CronJobs

Kubernetes supports different types of workloads depending on the application lifecycle, which can be broadly divided into long-running workloads and batch workloads.  
For long-running applications (such as web applications), Deployments are typically used.  
For jobs like batch processing, analytics or reporting, that are meant to live for a short period of time, performing specific tasks and then finish, the `Job` is the good choice.

> [!CAUTION]
>
> If you create a Pod instead of a Job to perform a short-lived task, like in tis example
>
> ```yaml
> apiVersion: v1
> kind: Pod
> metadata:
>   name: math-pod
> spec:
>   containers:
>   - name: math-add
>     image: ubuntu
>     command: ['expr', '3', '+', '2']
> ```
>
> you'll risk to trigger multiple restarts on the container because after it completes, Kubernetes continues to bring the container up again.  
> This is because the default restart policy for Pods is `Always`, which consists in restarting the container in an attempt to keep it running, until a threshold is reached.  
> This behaviour is defined by the property `spec.restartPolicy` which by default is set to always.  
> You can override this behaviour by setting this property to `Never` or `OnFailure`.  
> But anyway, this pattern is discouraged and we should define short-lived tasks in Jobs or CronJobs instead of Pods.

While `ReplicaSet` is used to make sure a specified number of pods is running at all time, a `Job` is used to run a set of pods to perform a given task to completion.

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: math-add-job
spec:
  template:
    spec:
      containers:
      - name: math-add
        image: ubuntu
        command: ['expr', '3', '+', '2']
      restartPolicy: Never
```

> [!NOTE]
>
> Jobs only support `restartPolicy: Never` or `OnFailure`.  
> `restartPolicy: Always` is not allowed.

```console
$ kubectl create -f job-definition.yaml
job.batch/math-add-job created

$ kubectl get jobs
NAME           STATUS    COMPLETIONS   DURATION   AGE
math-add-job   Running   0/1           6s         6s

$ kubectl get jobs
NAME           STATUS     COMPLETIONS   DURATION   AGE
math-add-job   Complete   1/1           14s        36s
```

We can see, also for the containers of a job, the standard output of their command with the command `kubectl logs <pod name>` or `kubectl logs job/<job name>`

```console
$ kubectl get pods
NAME                     READY   STATUS      RESTARTS        AGE
math-add-job-xxxbx       0/1     Completed   0               4m39

$ kubectl logs math-add-job-xxxbx
5
```

Finally, to delete the job run `kubectl delete job`

```console
$ kubectl delete job math-add-job
job.batch "math-add-job" deleted from default namespace
```

deleting the Job also deletes the Pods it created.

```console
$ kubectl get pods
No resources found in default namespace.
```

If the Job must complete successfully multiple times, set the `spec.completions` parameter.

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: math-add-job
spec:
  completions: 3
  template:
    spec:
      containers:
      - name: math-add
        image: ubuntu
        command: ['expr', '3', '+', '2']
      restartPolicy: Never
```

```console
$ kubectl create -f job-definition.yaml
job.batch/math-add-job created

$ kubectl get jobs
NAME           STATUS    COMPLETIONS   DURATION   AGE
math-add-job   Running   0/3           3s         3s

$ kubectl get jobs
NAME           STATUS    COMPLETIONS   DURATION   AGE
math-add-job   Running   1/3           7s         7s

$ kubectl get jobs
NAME           STATUS    COMPLETIONS   DURATION   AGE
math-add-job   Running   2/3           11s        11s

$ kubectl get jobs
NAME           STATUS     COMPLETIONS   DURATION   AGE
math-add-job   Complete   3/3           16s        16s

$ kubectl get pods
NAME                 READY   STATUS      RESTARTS   AGE
math-add-job-7g5g7   0/1     Completed   0          60s
math-add-job-mc2rh   0/1     Completed   0          65s
math-add-job-nls7g   0/1     Completed   0          70s
```

If completions is set to 3, Kubernetes waits until three Pods complete successfully.
Failed Pods do not count towards the completion count.  
If the pods fail (for example because the application exits with an error), the Job counts only successful Pod completions toward `.spec.completions` while failed Pods are retried until `.spec.backoffLimit` is reached.

There are situations where you want to fail a Job after some amount of retries due to a logical error in configuration etc.  
To do so, set `.spec.backoffLimit` to specify the number of retries before considering a Job as failed.  
Every failed Pod counts towards the `backoffLimit`. Once this limit is reached, the Job is marked as failed.  
The `.spec.backoffLimit` is set by default to 6.  
In this case (where the Pod randomly fails) we want to increase it in order to only count successful jobs.

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: random-error-job
spec:
  completions: 3
  backoffLimit: 30
  template:
    spec:
      containers:
      - name: random-error
        image: kodekloud/random-error
      restartPolicy: Never
```

```console
$ kubectl create -f job-definition.yaml
job.batch/random-error-job created

$ kubectl get job
NAME               STATUS     COMPLETIONS   DURATION   AGE
random-error-job   Complete   3/3           47s        63s

$ kubectl get pod
NAME                     READY   STATUS      RESTARTS   AGE
random-error-job-5kdkz   0/1     Error       0          48s
random-error-job-kzl9g   0/1     Completed   0          65s
random-error-job-ljwrd   0/1     Completed   0          23s
random-error-job-q7xkl   0/1     Completed   0          27s
random-error-job-xgbs5   0/1     Error       0          60s
```

By default the pods are created one after the other (the next pod is created only after the previous is finished).  
We can get the jobs created in parallel instead of sequentially with the property `spec.parallelism`.

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: random-error-job
spec:
  completions: 3
  parallelism: 3
  backoffLimit: 30
  template:
    spec:
      containers:
      - name: random-error
        image: kodekloud/random-error
      restartPolicy: Never
```

In this example, the job creates first 3 pods at once and then continues at rounds creating each time a number of pods equal to the missing number of completions.

```console
$ kubectl create -f job-definition.yaml
job.batch/random-error-job created

$ kubectl get pod
NAME                     READY   STATUS      RESTARTS   AGE
random-error-job-658nm   0/1     Completed   0          6s
random-error-job-q2nl6   0/1     Error       0          6s
random-error-job-vghnf   0/1     Error       0          6s

$ kubectl get job
NAME               STATUS    COMPLETIONS   DURATION   AGE
random-error-job   Running   1/3           12s        12s

$ kubectl get pod
NAME                     READY   STATUS      RESTARTS   AGE
random-error-job-658nm   0/1     Completed   0          13s
random-error-job-8qgbm   0/1     Completed   0          6s
random-error-job-pr8vb   0/1     Error       0          6s
random-error-job-q2nl6   0/1     Error       0          13s
random-error-job-vghnf   0/1     Error       0          13s

$ kubectl get job
NAME               STATUS    COMPLETIONS   DURATION   AGE
random-error-job   Running   2/3           18s        18s

$ kubectl get pod
NAME                     READY   STATUS      RESTARTS   AGE
random-error-job-658nm   0/1     Completed   0          26s
random-error-job-8qgbm   0/1     Completed   0          19s
random-error-job-jjzlj   0/1     Completed   0          6s
random-error-job-pr8vb   0/1     Error       0          19s
random-error-job-q2nl6   0/1     Error       0          26s
random-error-job-vghnf   0/1     Error       0          26s

$ kubectl get job
NAME               STATUS     COMPLETIONS   DURATION   AGE
random-error-job   Complete   3/3           25s        28s
```

Another common optional parameter for Jobs is `.spec.activeDeadlineSeconds`, which applies to the duration of the job, no matter how many Pods are created. Once a Job reaches this threshold, all of its running Pods are terminated and the Job status will become type: `Failed` with reason: `DeadlineExceeded`.

> [!NOTE]
>
> `parallelism` controls how many Pods may run at the same time.  
> `completions` controls how many successful Pod executions are required before the Job finishes.
>
> `spec.completions` and `spec.parallelism`, if omitted, are set to their default value, which is 1.

A CronJob is a Job that can be scheduled

```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: reporting-cron-job
spec:
  schedule: "*/1 * * * *"
  jobTemplate:
    spec:
      completions: 3
      parallelism: 3
      backoffLimit: 30
      template:
        spec:
          containers:
          - name: random-error
            image: kodekloud/random-error
          restartPolicy: Never
```

We can notice that the definition of a CronJob is more complex that the one of a Job: there are now three `spec` sections: one for the CronJob, one for the Job and one for the Pod.  
The `.spec.jobTemplate` defines a template for the Jobs that the CronJob creates, and it is required. It has exactly the same schema as a Job, except that it is nested and does not have an `apiVersion` or `kind`.  

> [!IMPORTANT]
>
> A CronJob creates a new Job according to the specified schedule.  
> Each Job then creates one or more Pods.
>
> ```text
> CronJob
>    │
>    ├── Job #1
>    │      └── Pod
>    │
>    ├── Job #2
>    │      └── Pod
>    │
>    └── Job #3
>           └── Pod
> ```

The schedule option is fundamental: it is a required string field in which we specify the time to be run using the Cron syntax.

```text
# ┌───────────── minute (0 - 59)
# │ ┌───────────── hour (0 - 23)
# │ │ ┌───────────── day of the month (1 - 31)
# │ │ │ ┌───────────── month (1 - 12)
# │ │ │ │ ┌───────────── day of the week (0 - 6) (Sunday to Saturday)
# │ │ │ │ │                                   OR sun, mon, tue, wed, thu, fri, sat
# │ │ │ │ │
# │ │ │ │ │
# * * * * *
```

For example, `0 3 * * 1` means this task is scheduled to run weekly on a Monday at 3 AM.

Other common optional parameters for CronJobs are:

- `.spec.concurrencyPolicy`: specifies how to treat concurrent executions of a Job that is created by this CronJob. The spec may specify only one of the following concurrency policies:
  - `Allow` (default): the CronJob allows concurrently running Jobs;
  - `Forbid`: the CronJob does not allow concurrent runs (if it is time for a new Job run and the previous Job run hasn't finished yet, the CronJob skips the new Job run);
  - `Replace`: if it is time for a new Job run and the previous Job run hasn't finished yet, the CronJob replaces the currently running Job run with a new Job run;
- `.spec.successfulJobsHistoryLimit`: specifies the number of successful finished jobs to keep. The default value is 3. Setting this field to 0 will not keep any successful jobs;
- `.spec.failedJobsHistoryLimit`: specifies the number of failed finished jobs to keep. The default value is 1. Setting this field to 0 will not keep any failed jobs;
- `.spec.startingDeadlineSeconds`: defines a deadline (in whole seconds) for starting the Job, if that Job misses its scheduled time for any reason;
- `.spec.suspend`: if you do set that field to `true`, all subsequent executions are suspended (they remain scheduled, but the CronJob controller does not start the Jobs to run the tasks) until you unsuspend the CronJob;
- `.spec.ttlSecondsAfterFinished`: it specifies how long Kubernetes keeps a completed Job before automatically deleting it (it will delete the Job cascadingly, i.e. delete its dependent objects, such as Pods, together with the Job).

| Resource   | Long running       | Executes once      | Scheduled          |
| ---------- | ------------------ | ------------------ | ------------------ |
| Pod        | :white_check_mark: | :x:                | :x:                |
| Deployment | :white_check_mark: | :x:                | :x:                |
| Job        | :x:                | :white_check_mark: | :x:                |
| CronJob    | :x:                | :white_check_mark: | :white_check_mark: |

## Network policies

### Description of Ingress and Egress in non Kubernetes environments

Network traffic can be viewed from two perspectives: **ingress** (incoming traffic) and **egress** (outgoing traffic).

For example, in a WebApplication that:

- receives client requests on the WebServer on port 80
- performs the request to the app Server exposing API on port 5000
- interacts with the database on port 3306

```text
            Ingress
               │ 80
               ▼
        +----------------+
        |   Web Server   |
        +----------------+
               │ 5000
         Egress│Ingress
               ▼
        +----------------+
        |   API Server   |
        +----------------+
               │ 3306
         Egress│Ingress
               ▼
        +----------------+
        |    Database    |
        +----------------+
```

Ingress and Egress traffic are divided as follows

- for the web server

  - the incoming traffic from the users on the default port 80 is ingress;
  - the outgoing request to the app server on the 5000 port is egress traffic;

- for the app that serves API:

  - the incoming traffic from the web server through the port 5000 is ingress;
  - the outgoing request to the database on port 3306 is egress traffic;

- for the Database perspective:

  - it receives ingress traffic on port 3306 from the API server.

If we should list the necessary rules to make this system work, we would have:

- an ingress rule that accepts HTTP traffic on port 80 on the webserver;
- an egress rule to allow traffic from the WebServer to port 5000 of the API server;
- an ingress rule to accept traffic on port 5000 on the API Server;
- an egress rule to allow traffic to port 3306 on the database server;
- an ingress rule on the DataBase server to accept traffic on the port 3306.

### Need of Network Policies for Kubernetes environments

In Kubernetes, each `Node`, `Pod` and `Service` has its own IP address.  
One of the prerequisites in Kubernetes is that, whatever solution is implemented, the pods should be able to communicate with each other without having to configure any additional settings (like routes).  
Pods are part of a flat network where, by default, every Pod can directly communicate with every other Pod without NAT, regardless of which node they run on.

> [!IMPORTANT]
>
> By default, Pods are **non-isolated**:
>
> - all ingress traffic is allowed;
> - all egress traffic is allowed.
>
> A Pod becomes isolated only when at least one NetworkPolicy selects it.

Coming back to the example made in the previous paragraph, in order to reach that architecture, we can:

- create a pod for the front-end, one for the API server and one for the database;
- create services to expose the pods.

but considering that by default all pods can talk to each other, in this way the front-end pod is able to communicate directly with the database pod.  
If we want to avoid that (maybe the security team or an audit required to prevent that to happening), that is where we would implement a **Network Policy** to allow traffic to the DB server only from the API server.

### Network Policies in Kubernetes environments

A `NetworkPolicy` is a namespaced Kubernetes resource that defines which network connections are allowed to and/or from a selected set of Pods.  
In order to apply a `NetworkPolicy` on a pod we use labels and selectors: we label the pod and we use the same label on the `podSelector` field in the `NetworkPolicy`.  
When building the rule, under `spec.policyTypes` we can specify whether the NetworkPolicy controls Ingress traffic, Egress traffic, or both.  
Finally, we define the allowed sources or destinations and, optionally, restrict them to specific ports and protocols.  
Once a Pod is selected by an Ingress `NetworkPolicy`, all ingress traffic is denied unless explicitly allowed by one or more matching `NetworkPolicy`.

> [!IMPORTANT]
>
> NetworkPolicies are additive.  
> If multiple NetworkPolicies select the same Pod, the allowed traffic is the union of all rules.
>
> Kubernetes does not evaluate NetworkPolicies in order.

In the case above, we would say "only allow ingress traffic from the API pod on the port 3306".  

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: db-policy
spec:
  podSelector:
    matchLabels:
      role: db
  policyTypes:
  - Ingress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          name: api-pod
    ports:
    - protocol: TCP
      port: 3306
```

> [!IMPORTANT]
>
> A NetworkPolicy is allow-based.  
> You specify what traffic is allowed. Everything else is denied for the selected Pods.

NetworkPolicies are enforced only if the cluster uses a **Container Network Interface** (CNI) plugin that supports them.  
Not all network solutions support NetworkPolicies. A few of them that support them are:

- Kube-router
- Calico
- Romana
- Weave Net

If we use Flannel for example, it does not currently support NetworkPolicies.  
Anyway, also in a cluster configured with a Network Solution that does not support NetworkPolicies, we can still create the policies, but they will just not be enforced.  
In other words, NetworkPolicies are defined by Kubernetes, but enforcement is delegated to the CNI plugin

Thinking about ingress and egress traffic, you could think that:

- the outgoing response of an incoming request;
- the incoming response of an outgoing request;

needs a different rule; however no additional rule is needed because NetworkPolicies are stateful: once a connection is allowed, the corresponding return traffic is automatically permitted.  
So when defining NetworkPolicies, you don't need to worry about the request responses.

There are additional selectors that can be used in `NetworkPolicies`, like

- `namespaceSelector`: defines from which namespace the traffic is allowed to reach the pod.

  ```yaml
      - namespaceSelector:
          matchLabels:
            kubernetes.io/metadata.name: prod
  ```

- `ipBlock`: defines a range of IP addresses from which allow traffic to the pod. It's commonly used to allow traffic from external IP addresses or external networks.

  ```yaml
      - ipBlock:
          cidr: 192.168.5.10/32
  ```

The same is valid for egress traffic.

| Selector          | Matches    |
| ----------------- | ---------- |
| podSelector       | Pods       |
| namespaceSelector | Namespaces |
| ipBlock           | IP ranges  |

Those rule selectors can be passed separately (as separate rules) or grouped together (as part of a single rule).

> [!IMPORTANT]
>
> Inside a single from (or to) entry, different selectors are combined with a logical AND.  
> Multiple entries in the from (or to) list are combined with a logical OR.

In the following example, we have two ingress elements / rules:

- the first rule states that traffic from pods labeled with `name: api-pod` in the `prod` namespace is allowed;
- the second rule states that traffic from ip ranges in the family `192.168.5.10/32` is allowed;

Traffic from sources meeting either of these criteria are allowed to pass through.

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: db-policy
spec:
  podSelector:
    matchLabels:
      role: db
  policyTypes:
  - Ingress
  - Egress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          name: api-pod
      namespaceSelector:
        matchLabels:
          kubernetes.io/metadata.name: prod
    - ipBlock:
        cidr: 192.168.5.10/32
    ports:
    - protocol: TCP
      port: 3306
  egress:
  - to:
    - ipBlock:
        cidr: 192.168.5.10/32
    ports:
    - protocol: TCP
      port: 80
```

In case you need to differentiate rules for ports or protocols, you can define multiple ingress or egress rules, each with its own `from` or `to` clause like in the following example.

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: internal-policy
  namespace: default
spec:
  podSelector:
    matchLabels:
      name: internal
  policyTypes:
  - Egress
  egress:
  - to:
    - podSelector:
        matchLabels:
          name: payroll
    ports:
    - protocol: TCP
      port: 8080
  - to:
    - podSelector:
        matchLabels:
          name: mysql
    ports:
    - protocol: TCP
      port: 3306
```

This is an example of NetworkPolicy that selects every Pod in the namespace and denies all incoming and outgoing traffic.

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-ingress
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  - Egress
```

> [!TIP]
>
> A NetworkPolicy never creates connectivity.
> It only restricts connectivity that already exists.  
> Pods must already be able to communicate through the CNI plugin.
>
> NetworkPolicies apply only to Pods.  
> They do **not** apply to:
>
> - Nodes
> - Services
> - Ingress resources

Useful commands:

- `kubectl get NetworkPolicies`: list all NetworkPolicies;
- `kubectl get networkpolicy`: same as above;
- `kubectl get networkpolicies`: same as above;
- `kubectl describe netpol <NetworkPolicy name>`: returns details about a specific NetworkPolicy.

## Ingress Networking

### Why do we need Ingress instead of exposing Services directly ?

With a Service, if we want to expose an application to the external world, we can use a Service of type NodePort.  
NodePort exposes the Service on a high TCP port (typically in the 30000–32767 range). Although functional, this is generally not suitable for production because users must access non-standard ports and additional infrastructure is usually required to provide a stable HTTP/HTTPS entry point.
In order to remediate, we could add a reverse proxy that accepts requests on port 80 and redirects them to the port exposed by the NodePort.  
The reverse proxy provides the public entry point, handles HTTP/HTTPS traffic and forwards requests to the NodePort Service.  
We can then point our DNS to this created server and finally users are able to access the application by simply visiting the FQDN.  
All this is possible if the application is hosted on-premise.

```text
                Internet
                   │
         www.my-online-store.com
                   │
          HTTP :80 / HTTPS :443
                   │
        +---------------------+
        |   Reverse Proxy     |
        |  (NGINX / HAProxy)  |
        +---------------------+
                   │
              NodeIP:30080
external infra     |
═══════════════════════════════════════
Kubernetes cluster |
                   │
           +--------------+
           | Service (NP) |
           |    :30080    |
           +--------------+
                   │
      ┌────────────┼────────────┐
      │            │            │
   +-------+   +-------+     +-------+
   | Pod 1 |   | Pod 2 |     | Pod 3 |
   +-------+   +-------+     +-------+
```

If instead we are on a public cloud environment, we use a service of type `LoadBalancer` instead of `NodePort`.  
The cloud provider automatically provisions an external load balancer that forwards traffic to the Service, typically through the NodePort automatically allocated for that Service.
The load balancer has an external IP that can be provided to allow users to access the application.  
We then set the DNS to point to this IP and users are finally able to access the application with the FQDN.

```text
               Internet
                   │
         www.my-online-store.com
                   │
          HTTP :80 / HTTPS :443
                   │
        +---------------------+
        | Cloud Load Balancer |
        | (AWS/GCP/Azure)     |
        +---------------------+
                   |
              NodeIP:30080
cloud provider     │
═══════════════════════════════════════
Kubernetes Cluster |
           +--------------+
           | Service (LB) |
           |    :30080    |
           +--------------+
                   │
      ┌────────────┼────────────┐
      │            │            │
   +-------+    +-------+    +-------+
   | Pod 1 |    | Pod 2 |    | Pod 3 |
   +-------+    +-------+    +-------+
```

If a new application in the same cluster needs to be exposed to the public on the same FQDN but a different URL, we then need:

- in case of on-premise: create a new Service of type `NodePort` and another reverse proxy listening on a different port;
- in case of cloud provider: create a new Service of type `LoadBalancer` which provisions another Load Balancer component set on a different port and witha different IP, which will result in increased costs.

If both applications must be exposed under the same FQDN, something must inspect the incoming URL and decide where to send the request.  
This requires an additional reverse proxy (or another load balancer) capable of URL-based routing.  
Every time a new application is deployed, this external component must be reconfigured.  
This means that routing configuration lives outside Kubernetes, making deployments harder to automate and maintain.

> [!TIP]
>
> The routing may also be based on the requested host (for example shop.example.com and api.example.com) instead of the URL path.

Finally, we also need to enable SSL for the applications, so the users can access the application using HTTPS.  
We can do this at different levels:

- application level
- load balancer / proxy-server level

Managing TLS inside every application would require every development team to implement and maintain certificates independently.  
Instead, it is preferable to terminate TLS at a single entry point, providing a consistent configuration for every application.

We also need to configure firewall rules for every externally exposed service.

```text
                         Internet
                             │
                  www.my-online-store.com
                             │
                    HTTP :80 / HTTPS :443
                             │
            +-----------------------------------+
            |        External Proxy / LB        |
            |       (URL-based Routing)         |
            |          /app1     /app2          |
            +-----------------------------------+
                   │                      │
            NodeIP:30080           NodeIP:30082
cloud provider     │                      │
══════════════════════════════════════════════════════════════
Kubernetes Cluster |                      |
          +----------------+      +----------------+
          | Service 1 (LB) |      | Service 2 (LB) |
          |     :30080     |      |     :30082     |
          +----------------+      +----------------+
                   │                        │
             ┌─────┴─────┐            ┌─────┴─────┐
             │           │            │           │
         +-------+   +-------+    +-------+   +-------+
         | Pod 1 |   | Pod 2 |    | Pod 1 |   | Pod 2 |
         +-------+   +-------+    +-------+   +-------+
```

Every new application requires:

- another Service;
- another externally exposed endpoint (LoadBalancer or reverse proxy);
- additional firewall rules;
- SSL configuration;
- DNS changes;
- proxy configuration updates.

All of this will become difficult to manage when we scale the application.  
Kubernetes addresses these challenges by introducing the `Ingress` resource, which centralizes external access configuration.

The Ingress Controller becomes the **single entry point for the cluster**. Routing, SSL termination and virtual hosting are managed through Kubernetes Ingress resources instead of external infrastructure.

```text
                         Internet
                             │
                  www.my-online-store.com
                             │
                    HTTP :80 / HTTPS :443
                             │
═══════════════════════════════════════════════════════════
Kubernetes Cluster           |
                  +--------------------+
                  | Service (NP or LB) |
                  +--------------------+
                             │
            +-----------------------------------+
            |        Ingress Controller         |
            +-----------------------------------+
            /app1  |                  /app2 |
          +-----------------+      +-----------------+
          | Service 1 (CIP) |      | Service 2 (CIP) |
          +-----------------+      +-----------------+
                   │                        │
             ┌─────┴─────┐            ┌─────┴─────┐
             │           │            │           │
         +-------+   +-------+    +-------+   +-------+
         | Pod 1 |   | Pod 2 |    | Pod 1 |   | Pod 2 |
         +-------+   +-------+    +-------+   +-------+
```

> [!NOTE]
>
> An Ingress is not directly exposed to the Internet but needs a Service.

| Without Ingress                     | With Ingress                            |
| ----------------------------------- | --------------------------------------- |
| Multiple LoadBalancers              | Single LoadBalancer                     |
| Multiple public IPs                 | Single public IP                        |
| Proxy configured outside Kubernetes | Routing defined in Kubernetes manifests |
| SSL configured separately           | Centralized SSL termination             |
| DNS/proxy changes for every app     | Only update the Ingress manifest        |
| URL routing configured externally   | URL routing managed by Kubernetes       |

> [!NOTE]
>
> An Ingress resource is only a set of routing rules stored in Kubernetes and does not receive network traffic by itself.  
> An Ingress Controller (such as NGINX Ingress Controller or Traefik) is the component responsible for implementing those rules.

### Ingress and Ingress Controller

Think of ingress as a Layer 7 LoadBalancer built into the Kubernetes cluster that can be configured using native Kubernetes primitives.

In order to implement Ingresses, you need two elements

- **Ingress Controller**: deploy a reverse proxy or a load balancing solution on the Kubernetes Cluster;
- **Ingress resources**: define the configuration (routes, cerrtificats, etc) in `Ingress` Kubernetes manifests.

> [!NOTE]
>
> If you simply create Ingress resources (without deploying an Ingress controller) they won't work.  
> A Kubernetes cluster does not come with an Ingress Controller by default.

There are many available solutions for Ingress controller like NGINX, HAPROXY, Traefik, GCE (Google Layer 7 Load Balancer), Contour.  
Out of these, NGINX and GCE are currently being supported and maintained by the Kubernetes project.

Keep in mind that the load balancer component is just a part of Ingress Controllers.  
Ingress Controllers have additional intelligence to monitor the Kubernetes Cluster for definitions of ingress resources and configure the underlying proxy accordingly.

### Simplified architecture of an Ingress Controller

> [!NOTE]
>
> Modern Kubernetes clusters usually install Ingress Controllers using Helm charts or vendor-provided manifests.  
> The following Deployment is intentionally simplified to illustrate the components involved.

In the following example we'll use NGINX as Ingress Controller.  
NGINX controller is deployed just as another Deployment in Kubernetes.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-ingress-controller
spec:
  replicas: 1
  selector:
    matchLabels:
      name: nginx-ingress
  template:
    metadata:
      labels:
        app: nginx-ingress
    spec:
      containers:
      - name: nginx-ingress-controller
        image: registry.k8s.io/ingress-nginx/controller:v1.13.1
        args:
        - /nginx-ingress-controller
        - --configmap=$(POD_NAMESPACE)/nginx-configuration
        env:
        - name: POD_NAME
          valueFrom:
            fieldRef:
              fieldPath: metadata.name
        - name: POD_NAMESPACE
          valueFrom:
            fieldRef:
              fieldPath: metadata.namespace
        ports:
        - name: http
          containerPort: 80
        - name: https
          containerPort: 443
```

We also passed two environment variables, carrying the Pod name and namespace.  
The NGINX service requires them to read the configuration data between the pod.

Finally specify the ports used by the Ingress Controller which are `80` and `443`.

We also add an empty ConfigMap to pass future values (like `err-log-path`, `keep-alive` and `ssl-protocols`)

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: nginx-configuration
```

In order to expose the Ingress Controller externally, we add a `Service` of type `NodePort` with the selector pointing to the label of the pod.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-ingress
spec:
  type: NodePort
  ports:
  - port: 80
    targetPort: 80
    protocol: TCP
    name: http
  - port: 443
    targetPort: 443
    protocol: TCP
    name: https
  selector:
    name: nginx-ingress
```

As said before, Ingress Controller can monitor the cluster and configure the underlying NGINX server when something changes.  
But in order for the Incress Controller to do this, we need a `ServiceAccount` with the right set of permissions defined with `Roles`, `ClusterRoles` and `RoleBindings`.

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: nginx-ingress-serviceaccount
```

```bash
kubectl create -f nginx-ingress-controller.yaml
kubectl create -f nginx-configuration.yaml
kubectl create -f nginx-ingress.yaml
kubectl create -f nginx-ingress-serviceaccount.yaml
```

### Ingress resources

Let's define now some Ingress rules on the Ingress Controller via `Ingress` resources.

You can create Ingress rules to, for example:

- simply forward Ingress traffic to a single application;
- route traffic to different applications based on URLs,
- route traffic to different applications based on the domain name (FQDN).

In the first case, the section `defaultBackend` defines where the traffic is routed to.  
Considering that it's a single backend, we don't need any specific rule; simply specifying service name and port is enough.

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: ingress-wear
spec:
  ingressClassName: nginx
  defaultBackend:
    service:
      name: wear-service
      port:
        number: 80
```

```bash
kubectl create -f ingress-wear.yaml
```

Now we can retrieve the created Ingress

```console
$ kubectl get ingress
NAME           CLASS    HOSTS   ADDRESS   PORTS   AGE
ingress-wear   <none>   *                 80      32s
```

If instead, we want to create different rules based on different URLs, we can do the following

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: ingress-wear-watch
spec:
  ingressClassName: nginx
  rules:
  - http:
      paths:
      - path: /wear
        pathType: Prefix
        backend:
          service:
            name: wear-service
            port:
              number: 80
      - path: /watch
        pathType: Prefix
        backend:
          service:
            name: watch-service
            port:
              number: 80
```

The available path types are the following

| pathType               | Meaning              |
| ---------------------- | -------------------- |
| Exact                  | Exact URL            |
| Prefix                 | Prefix matching      |
| ImplementationSpecific | Controller-dependent |

```bash
kubectl create -f ingress-wear-watch.yaml
```

We now see two backend URLs under the `Rules` section.

```console
$ kubectl describe ingress ingress-wear-watch
Name:             ingress-wear-watch
Labels:           <none>
Namespace:        default
Address:
Ingress Class:    <none>
Default backend:  <default>
Rules:
  Host        Path  Backends
  ----        ----  --------
  *
              /wear    wear-service:80 (<error: services "wear-service" not found>)
              /watch   watch-service:80 (<error: services "watch-service" not found>)
Annotations:  <none>
Events:       <none>
```

You can also still notice that the `Default backend` field is empty.  
We can specify the `defaultBackend` which is the redirect applied when the URL hit does not match any of the rules.  
If omitted, unmatched requests typically receive a 404 response generated by the Ingress Controller.

The third of configuration is using domain names or host names.  
In this case we create 2 rules (one for each domain) and we add the field `host` to each rule.

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: ingress-wear-watch
spec:
  ingressClassName: nginx
  rules:
  - host: wear.my-online-store.com
    http:
      paths:
      - path: /wear
        pathType: Prefix
        backend:
          service:
            name: wear-service
            port:
              number: 80
  - host: watch.my-online-store.com
    http:
      paths:
      - path: /watch
        pathType: Prefix
        backend:
          service:
            name: watch-service
            port:
              number: 80
```

> [!CAUTION]
>
> URL-based routing usually uses a single rule containing multiple paths.
> Host-based routing instead uses one rule for each host.

You can also add TLS on an ingress

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: tls-example-ingress
spec:
  tls:
  - hosts:
      - https-example.foo.com
    secretName: testsecret-tls
  rules:
  - host: https-example.foo.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: service1
            port:
              number: 80
```

and the certificate is retrieved from a Secret that must contain the fields `tls.crt` and `tls.key`

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: testsecret-tls
  namespace: default
data:
  tls.crt: base64 encoded cert
  tls.key: base64 encoded key
type: kubernetes.io/tls
```

### IngressClass

In modern Kubernetes clusters, multiple Ingress Controllers may coexist.

For example:

- NGINX
- Traefik
- HAProxy

Each controller watches only the Ingress resources that belong to its own class.

An `IngressClass` is a cluster-scoped resource that identifies which Ingress Controller should implement a given Ingress resource.  
An `Ingress` references it through the field

```yaml
spec:
  ingressClassName: nginx
```

for example

```yaml
apiVersion: networking.k8s.io/v1
kind: IngressClass
metadata:
  name: nginx
spec:
  controller: k8s.io/ingress-nginx
```

The corresponding Ingress than becomes

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: app-ingress
spec:
  ingressClassName: nginx
  rules:
  ...
```

```text
           Ingress
  ingressClassName: nginx
             │
             ▼
     +----------------+
     | IngressClass   |
     |     nginx      |
     +----------------+
             │
             ▼
+---------------------------+
| NGINX Ingress Controller  |
+---------------------------+
```

If an `Ingress` does not specify an `ingressClassName`, the cluster may:

- assign the default IngressClass;
- ignore the Ingress if no default exists.

This depends on how the Ingress Controllers are configured.

> [!NOTE]
>
> Before Kubernetes v1.18 it was common to specify the controller through the annotation
>
> ```yaml
> kubernetes.io/ingress.class: nginx
> ```
>
> This annotation is now deprecated in favour of the ingressClassName field.

### Additional options

Different ingress controllers have different options that can be used to customise the way it works.

For example, [NGINX Ingress controller has many options](https://kubernetes.github.io/ingress-nginx/examples/) that can be passed as environment variables or annotations.

One useful argument to pass to the NGINX Ingress Controller allows the controller to route unmatched requests to a default `Service`, even if no default backend is defined in the `Ingress` resource.

```yaml
        args:
        - --default-backend-service=<namespace>/<service>
```

While some common examples of Annotations for the NGINX Ingress Controller are:

| Annotation                                    | Purpose                                                               |
| --------------------------------------------- | --------------------------------------------------------------------- |
| `nginx.ingress.kubernetes.io/rewrite-target`  | Rewrites the request path before forwarding it to the backend Service |
| `nginx.ingress.kubernetes.io/ssl-redirect`    | Redirects HTTP requests to HTTPS                                      |
| `nginx.ingress.kubernetes.io/proxy-body-size` | Sets the maximum accepted request body size                           |

These annotations are specific to the NGINX Ingress Controller. Other Ingress Controllers may use different annotations or configuration mechanisms.

Useful commands:

- `kubectl create ingress <ingress-name> --rule="host/path=service:port"`: template for creating an Ingress imperatively;
- `kubectl create ingress ingress-test --rule="wear.my-online-store.com/wear*=wear-service:80"`: creates an Ingress that routes requests for `wear.my-online-store.com` whose path starts with `/wear` to the `wear-service` Service on port 80.

## Info about the CKAD (Certified Kubernetes Application Developer) certification exam

The exam lasts 2 hours and typically includes around 15–20 performance-based tasks.

You are allowed to consult the official Kubernetes documentation and the other resources explicitly permitted during the exam.

All the questions have their weight displayed.  
There are some questions with lower weight that will consume more time. So its better to come back to these questions once you are done with questions of higher weight.

Read the instructions given before the start of the exam on how to copy and paste from kubernetes.io documentation into the shell.

You will be provided with a Notepad.  
Make use of it; for example you can note the number of the questions that you want to come back later and their weight.

Proctors can be very strict about exam code of conduct.

Before you attempt a new question, be sure to execute the command to switch to the correct Kubernetes context. It is given at the start of the question.  
Each question specifies the Kubernetes context to use; always switch to the requested context before making any changes.

Sometimes you need to ssh to another node or change to root user. Beware of where you are all the time on bash terminal.

Other suggestions for the exam:

- you should learn well the commands for service creation and become generally confident with the section "Imperative commands", so repeat <https://aruba.udemy.com/course/certified-kubernetes-application-developer/learn/lecture/15021564#overview> (<https://uklabs.kodekloud.com/topic/imperative-commands/>) until you are reliable and fast;
- understand the differences between Ingress, Gateway, LoadBalancer Service and when each should be used;
- learn the syntax for creating and consuming ConfigMaps and Secrets;
- become fast and confident with both imperative commands and declarative syntax;
- consult <https://github.com/lucassha/CKAD-resources> and train well on <https://github.com/dgkanatsios/CKAD-exercises>;
- if appropriate, you can also use the book "Kubernetes Up and Running - by Kelsey Hightower, Brendan Burns and Joe Beda";
- do not waste time writing manifests from scratch, use `kubectl create ... --dry-run=client -o yaml` whenever possible, then edit the generated manifest;
- if you are stuck on a question, move on and come back later; it is usually better to complete several easier questions than spend too much time on a single difficult one.
