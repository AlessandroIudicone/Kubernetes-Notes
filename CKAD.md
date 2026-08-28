# CKAD Certification

---

## Architecture

A Node is a machine (physical or virtual) and can be:

- worker node: where applications run;
- master node: responsible for managing the cluster and orchestrating the nodes.

A cluster is a set of nodes grouped together.

The components of a Kubernetes cluster are the following:

| component               | on node | description                                                                                                                    |
| ----------------------- | ------- | ------------------------------------------------------------------------------------------------------------------------------ |
| kube-apiserver          | master  | (also called the API Server) acts as the front-end. Users, CLI, devices talk to this component to interact with the cluster    |
| etcd                    | master  | is a distributed and reliable key-value store used by Kubernetes to store the data to manage the cluster                       |
| kube-scheduler          | master  | distributes workloads across the worker nodes                                                                                  |
| kube-controller-manager | master  | the brain behind orchestration, responsible for monitoring the cluster state and reconciling it with the desired state         |
| Container runtime       | worker  | is the underlying software used to run containers (it can be containerd or CRI-O or Docker Engine¹)                            |
| kubelet                 | worker  | the agent that runs on each node in the cluster, which is responsible for ensuring that the containers are running as expected |
| kube-proxy              | worker  | configures the networking rules that allow Services to reach the appropriate Pods                                              |

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

> [!TIP]
>
> The Control Plane is the software "brain" of a Kubernetes cluster that makes decisions and manages state.  
> The Master Node is the physical or virtual server (or servers) where those Control Plane software components are actually running.  
> Think of the Control Plane as the mind, and the Master Node as the physical body that houses it.

There is also an optional component for the Control Plane, the `cloud-controller-manager`, which integrates with underlying cloud provider(s).

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
> Dockershim     cri-dockerd
>     │              │
> Docker Engine  Docker Engine
>     │              │
> containerd     containerd
>     │              │
>   runc           runc
> ```

Then, the command line tool is the `kubectl` (also read as "Kube Control", as a way to remember what it stands for).  
Useful `kubectl` commands are:

- `kubectl cluster-info`: show information about the cluster;
- `kubectl describe`: show details of a specific resource or group of resources;
- `kubectl create -f <file-name>`: creates the resources specified inside the file;
- `kubectl replace --force -f <filename>`: deletes and recreates the resources specified inside the file;
- `kubectl get all --no-headers`: list all the objects created in the default namespace, without printing the header line;
- `kubectl get all -A`: list all the objects in all namespaces;
- `kubectl <command> [subcommand] --help`: gives information about the current command and subcommand, including the different available parameters;
- `kubectl edit <resource> <resource name>`: edit the manifest of a resource on-the-fly;
- `kubectl <verb> <resource> --no-headers | wc -l`: count the number of objects returned.

The configuration file of a Kubernetes object (an instance of a given `Kind`) is called a "manifest".

---

## Pods

The structure of a Pod YAML manifest is the following

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
> It does **not** open a port and Kubernetes does not use it to forward traffic.  
> The application inside the container decides which port to listen on.
>
> Instead, a Service must forward traffic (`targetPort`) to the port where the application is actually listening, regardless of whether `containerPort` is declared.

In a Pod manifest, you can also optionally specify the `command` and `args` fields for each container.  
The first overrides the default command (defined in the image) while the second overrides the arguments of the command.  
For comparison with some container runtimes (like Docker):

- using `command` is like specifying the `ENTRYPOINT` instruction in the Dockerfile or `--entrypoint` in the CLI;
- using `args` is like specifying the `CMD` instruction in the Dockerfile or the arguments passed by the CLI.

`command` overrides the Docker image's `ENTRYPOINT` while `args` overrides the Docker image's `CMD`.  
If only `args` is specified, Kubernetes keeps the image's `ENTRYPOINT` and replaces only the `CMD`.

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
kubectl run nginx --image=nginx --command -- <COMMAND> [arg1] .. [argn]
```

You can also specify environment variables inside a pod, with the `env` property.  
`env` is an array, so every element starts with a dash indicating that it is an item inside the array.  
Each item has `name` and `value` properties.

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

However, there are other ways of setting environment variables in Kubernetes, such as injecting them through ConfigMaps and Secrets, which we'll see later.

Useful commands:

- `kubectl run -n finance redis --image=redis --restart=Never`: create a pod named redis, with the redis image, in the namespace "finance", with restart policy set to `Never`;
- `kubectl get pod <pod-name> -o wide`: kubectl get pod but with output in the plain-text format with additional information (like the node name);
- `kubectl get pod <pod-name> -o yaml > pod-definition.yaml`: outputs the current configuration of a pod (but can be used for objects in general) to a YAML file;
- `kubectl get pods -w`: list pods and watch for changes;
- `kubectl run httpd --image=httpd:alpine --port=80`: creates an httpd pod and declares port 80 as `containerPort`;
- `kubectl run httpd --image=httpd:alpine --port=80 --expose=true`: in addition to the command above, it automatically also creates a `Service` of type `ClusterIP` that exposes port 80 inside the cluster; a DNS entry is then automatically created for it, allowing other pods to resolve it.

---

## ReplicaSets

This is the structure of a `ReplicaSet` YAML manifest

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

In the ReplicaSet, `.spec.template.metadata.labels` must match `spec.selector`, or it will be rejected by the API.

> [!TIP]
>
> In the past, the `ReplicationController` (introduced in the initial Kubernetes v1.0 release in July 2015) was used instead of `ReplicaSet` (introduced later in Kubernetes v1.2 in March 2016).  
> The main difference between the `ReplicationController` (deprecated) and the `ReplicaSet` is that the field `selector` became mandatory in the second (in case omitted, in the former `ReplicationController`, Kubernetes assumes that `selector` is the same as the `labels` field).

There are three ways to scale a ReplicaSet:

- `kubectl apply -f replicaset-definition.yaml`: modify the value of `replicas` inside the file manifest and then `apply` it;
- `kubectl scale --replicas=6 -f replicaset-definition.yaml`: uses an existing manifest file (still required as input) but does not update its content;
- `kubectl scale --replicas=6 replicaset frontend`: this is an imperative command that scales the already deployed ReplicaSet, without the need for a manifest file.

There are also options to automatically scale the replicas based on the load, but we'll see them in another course.

> [!WARNING]
>
> Updating the Pod template of a `ReplicaSet` does not trigger an automatic rolling update.  
> The updated Pod template is used only for newly created Pods.  
> Existing Pods are not automatically replaced and continue running with their current configuration until they are deleted.  
> Deployments should be preferred when controlled rolling updates or rollbacks are required.

Useful commands:

- `kubectl create -f replicaset-definition.yaml`: creates a `ReplicaSet` starting from the definition inside the YAML manifest;
- `kubectl get replicaset <replicasetname>`: retrieves the `ReplicaSet` in the default namespace;
- `kubectl delete replicaset <replicasetname>`: deletes a `ReplicaSet` and all of its pods;
- `kubectl replace -f replicaset-definition.yaml`: replaces or updates a `ReplicaSet`;
- `kubectl scale`: scales the replicas of a `ReplicaSet`.

---

## Deployments

The `Deployment` is "higher in hierarchy" compared to `ReplicaSet` (and `ReplicationController`).  
If the RS is a controller of Pods, a `Deployment` manages one or more ReplicaSets, providing rollout and rollback capabilities.

The structure of a `Deployment` YAML manifest is almost identical to the one of the `ReplicaSet`, except for the Kind.

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

At creation, the `Deployment` automatically generates a `ReplicaSet` as well (which will be named after the `Deployment`), in addition to the `Deployment` object itself.

> [!IMPORTANT]
>
> Remember that you cannot edit specifications of an existing `Pod` other than:
>
> - `spec.containers[*].image`
> - `spec.initContainers[*].image`
> - `spec.activeDeadlineSeconds`
> - `spec.tolerations`
>
> If you really need to, you can instead recreate the pod with `kubectl replace --force -f <filename>`.
>
> With `Deployments`, instead, you can easily edit any field / property of the Pod template.  
> Since the pod template is a child of the `Deployment` specification, with every change to the Pod template, the Deployment automatically creates the necessary new Pods (via a new `ReplicaSet`) to replace the old ones.  
> So if you need to edit a property of a `Pod` that is part of a `Deployment`, you may do that simply by running the command `kubectl edit deployment <deployment name>`.

Useful commands:

- `kubectl scale deployment nginx --replicas=4`: scale a `Deployment` using the kubectl scale command;
- `kubectl create deployment <deployment name> --image=<image>`: create a `Deployment`;
- `kubectl get deployment <deployment name>`: retrieves a `Deployment` based on its name;
- `kubectl get deployments`: retrieves all deployments from the default namespace;
- `kubectl edit deployment <deployment name>`: edit a property of a `Deployment`.

---

## Namespaces

The `Namespace` is the logical segmentation of Kubernetes clusters.  
Many kinds (like `Pod`) only exist inside a namespace and in this case we say that the resource is **namespaced**; other resources are not limited to namespaces, like namespaces themselves (they are reachable from the whole cluster).  
Each namespace can have its own set of policies defining who can do what, their quotas (number of CPUs, GB of RAM), limits, etc.

When a Kubernetes cluster is created, it creates:

- the default `Namespace`, which is normally named `default`;
- a system namespace named `kube-system`, in which a series of pods and services are created for serving the cluster, isolating them from the user and preventing them from being modified or deleted;
- a third, automatically created namespace named `kube-public`, which is readable by all users, including unauthenticated users in many cluster configurations (this namespace is a good place to put resources that should be available to all users);
- a fourth namespace, `kube-node-lease`, which holds the Lease objects used for node heartbeats, helping the control plane detect node failures more efficiently.

Inside the same namespace, the resources can refer to each other simply using their service name (as if they were people living inside the same house). In this way, for example, the `Pod` named `webapp` can reach the `Service` named `db-service` which resides in the same `Namespace` simply using its name, like for example with `mysql.connect("db-service")`.  
Instead, if an application would like to reach a service in the same cluster but in another namespace, it should use the syntax `<Service>.<Namespace>.svc.cluster.local`, so for example, in case we want to reach a mysql db on the service `db-service` in the namespace `dev`, the application should use the syntax `mysql.connect("db-service.dev.svc.cluster.local")`.  
This happens because:

- when a service is created, a DNS entry is automatically added;
- `cluster.local` is the default domain of a Kubernetes cluster;
- `svc` stands for `Service`.

The `Namespace` can also be specified inside the `metadata` section of the various manifest files described above.  
There is also an option to show the resources in all namespaces, which is `--all-namespaces` and the shorthand is `-A`.

Useful commands:

- `kubectl get pod`: retrieves all the pods in the default namespace;
- `kubectl get pod --all-namespaces`: retrieves all the pods in all the namespaces;
- `kubectl get service -A`: retrieves all the services in all the namespaces;
- `kubectl get service -n [namespace name]`: retrieves all the services in a specific namespace.

---

## Services

Services are Kubernetes objects that allow communication between the workloads and with external entities.  
Services allow loose coupling by abstracting away the physical addresses of the workloads.  

> [!NOTE]
>
> A `Service` can select Pods running on different nodes, but Kubernetes does not create one Service per node.
>
> A `Service` is **NOT** replicated across multiple nodes but is a namespaced Kubernetes object.
>
> Instead, the `Service` maintains an object called `Endpoints` with a list of endpoints corresponding to all matching Pods, regardless of the node where they run.
>
> Networking components such as kube-proxy (or eBPF-based implementations like Cilium) configure the forwarding rules on the cluster nodes.

There are mainly 3 types of services:

- `ClusterIP`: creates a virtual IP inside the cluster to enable communication between services. It stays inside the Kubernetes network;
- `NodePort`: exposes a specific port on every node and forwards incoming requests to the selected Pods;
- `LoadBalancer`: provisions a load balancer in supported cloud providers (to distribute the load). It is normally used for exposing the pods on public networks.

Most Services are assigned a virtual IP address called ClusterIP; Headless Services are an exception and are created with `clusterIP: None`.

In Services, the `selector` defines which Pods the Service targets.  
To link a `Service` with the pods, the content of `selector` in the former must match the `labels` defined in the latter.  
A `Service` exposes, through the correct use of the selectors:

- a single pod on a single node;
- multiple pods on a single node;
- multiple pods on multiple nodes.

This is because every `Service` load-balances traffic across the matching Pods (via kube-proxy or an equivalent networking component), regardless of its type.

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

`ClusterIP` is the default type of `Service`; if the type is not declared, the `Service` is automatically created with the `ClusterIP` type.  
Once created, the `ClusterIP` Service allows access to other pods both using the CLUSTER-IP (number) assigned and its name.

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
      targetPort: 80
      nodePort: 30007
```

In the `Service` of type `NodePort`, in each element of the array `ports`, along with `port` (port of the Service itself) and `targetPort` (destination port on the pod), there is the `NodePort`, which is the port exposed outside the Kubernetes cluster.  
The `NodePort` must always be inside the range 30000 - 32767; it is an optional field and if not declared, it will be automatically allocated a free port from that range.  

> [!NOTE]
>
> A `Service` can map any incoming `port` to a `targetPort`. By default and for convenience, the `targetPort` is set to the same value as the port field.

Once this type of service has been created, we can access the exposed pod through `curl http://<node_ip>:<nodePort>`.  

`NodePort` directly exposes a port on every cluster node.  
The use cases of the `NodePort` `Service` are mainly debugging, temporary operations, and deployments on non-sensitive environments.

With `NodePort`, the client must call the IP address of one of the nodes together with the `nodePort`. This has two main disadvantages:

- if nodes are replaced / updated / added, they may be assigned a new IP address and this breaks the pattern of the previous calls;
- you declare the IP address of the Kubernetes nodes, and this may raise security concerns.

For production environments, it is often preferable to place a `LoadBalancer` or an `Ingress` in front of applications to provide more flexibility and traffic management features.  

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

The `Service` types can be viewed as incremental extensions:

- `ClusterIP` exposes the Service inside the cluster through a virtual IP;
- `NodePort` includes all the features of `ClusterIP` and additionally exposes the `Service` on a fixed port on every cluster node;
- `LoadBalancer` extends a `NodePort` `Service` by requesting the underlying infrastructure (typically a cloud provider) to provision an external load balancer that forwards traffic to the Service's `nodePort`.

Sometimes you don't need load-balancing and a single Service IP.  
In this case, you can create a Headless `Service`, by explicitly specifying

```yaml
spec:
  clusterIP: None
```

With the headless `Service`, DNS queries return the IP addresses of the individual pods behind the `Service`.  
This is commonly used by `StatefulSets`.  
So the headless `Service` lets you communicate with an exact pod instead of landing on a random one chosen by the ClusterIP.  
A use case for this type of service is when you have to deal with a stateful application; for example a database, when there is a primary node that allows writing and many worker nodes that allow only reading, and you need to choose which one to connect to based on whether you need to write or read data.

For all types of `Service`:

- inside `spec` we define the array `ports`. If we map only one port, the field `name` is optional; if we need to map two or more ports, the field `name` becomes mandatory;
- when a `Service` is created, Kubernetes also automatically creates an object called `Endpoints` with the same name as the `Service`, whose role is to keep track of which pods are the current endpoints of the service. This list is automatically updated whenever matching pods are created, deleted, or rescheduled (for example when they are restarted, scaled, or replaced), so that newly created pods are added as endpoints and pods that no longer exist are removed;
- the `targetPort` should match the port on which the application inside the container is actually listening;
- when creating a `Service`, an IP address (called the Service's CLUSTER-IP) is assigned to it, allowing access from other workloads using either the assigned CLUSTER-IP or its name.

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

---

## Imperative commands

While you would be working mostly using the declarative commands (using definition files), imperative commands can help in getting one-time tasks done quickly, as well as generating a definition template easily.

> [!TIP]
>
> This would help save a considerable amount of time during your exams.  

Familiarize yourself with the two options that can come in handy while working with the below commands:

- `--dry-run`: by default, as soon as the command is run, the resource will be created. If you simply want to test your command, use the `--dry-run=client` option. This will not create the resource and instead will simulate the creation, telling whether the resource can be created and if the command is right;
- `-o yaml`: this will output the resource definition in YAML format on the screen.

Use the above two in combination along with Linux output redirection to generate a resource definition file quickly that you can then modify and create resources as required, instead of creating the files from scratch.

Here are some examples

| Command                                                      | Description                         | Dry-run command to generate YAML manifest (and eventually redirect it to a file)                 |
| ------------------------------------------------------------ | ----------------------------------- | ------------------------------------------------------------------------------------------------ |
| `kubectl run nginx --image=nginx`                            | Create an NGINX Pod                 | `kubectl run nginx --image=nginx --dry-run=client -o yaml > nginx-pod.yaml`                      |
| `kubectl create deployment nginx --image=nginx`              | Create a deployment                 | `kubectl create deployment nginx --image=nginx --dry-run=client -o yaml`                         |
| `kubectl create deployment nginx --image=nginx --replicas=4` | Create a deployment with 4 Replicas | `kubectl create deployment nginx --image=nginx --dry-run=client -o yaml > nginx-deployment.yaml` |

The following command creates a `Service` named redis-service of type ClusterIP to expose the pod redis on port 6379, via the `expose` command

```bash
kubectl expose pod redis --port=6379 --name redis-service --dry-run=client -o yaml
```

and this will automatically use the pod's labels as selectors.  
Or instead, you can do the same via the service creation command

```bash
kubectl create service clusterip redis --tcp=6379:6379 --dry-run=client -o yaml
```

but this will not use the pod labels as selectors; instead it will assume the selector `app=redis`.  
Considering that you cannot pass in selectors as an option, it does not work well if your pod has a different label set.  
So with this second option, after having generated the file, you could need to modify the selectors before creating the service.

To create a `Service` named nginx of type `NodePort` that exposes pod nginx's port 80 as port 30080 on the nodes, using the `expose` command, you need the following command

```bash
kubectl expose pod nginx --port=80 --name nginx-service --type=NodePort --dry-run=client -o yaml
```

to generate the definition file.  
But considering that this will automatically use the pod's labels as selectors and we cannot specify the `nodePort`, we then need to add the `nodePort` in the file manually before creating the service.  
Or instead, you can do similarly via the service creation command

```bash
kubectl create service nodeport nginx --tcp=80:80 --node-port=30080 --dry-run=client -o yaml
```

But this will not use the pods' labels as selectors and you should still modify the manifest before applying it.

Both the above commands have their own challenges.  
While one of them cannot accept a selector, the other cannot accept a `nodePort`.  
I would recommend going with the `kubectl expose` command, and then manually add the `nodePort` in the manifest file before creating the service.

---

## Formatting Output with kubectl

The default output format for all kubectl commands is the human-readable plain-text format.  
The -o flag lets you output the details in several different formats.

```bash
kubectl [command] [TYPE] [NAME] -o <output_format>
```

Here are some of the commonly used formats:

- `-o json`: output a JSON formatted API object;
- `-o name`: print only the resource name and nothing else;
- `-o wide`: output in the plain-text format with additional information;
- `-o yaml`: output a YAML formatted API object.

---

## ConfigMaps

Instead of defining environment variables in the pod definition files, we can inject them into the containers of the pods using ConfigMaps.  
ConfigMaps are used to pass configuration data in the form of key-value pairs.

There are two phases involved in setting ConfigMaps:

1) create the `ConfigMap`;
2) inject it into the pod.

Like other Kubernetes objects, we can create the `ConfigMap` in two ways:

- imperative: `kubectl create configmap` specifying the required arguments;
- declarative: `kubectl create -f [file name]`.

Imperatively, this can be done by declaring the key values in the `kubectl` command

```bash
kubectl create configmap <configmap name> --from-literal=key=value <--from-literal=key=value>
kubectl create configmap app-config --from-literal=APP_COLOR=blue --from-literal=APP_ENV=prod
```

or (still imperatively) through a file (not a manifest)

```bash
kubectl create configmap <configmap name> --from-file=path-to-file
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

It is important to name the ConfigMaps appropriately as we'll use these names later to associate them with the pods.

In order to inject the `ConfigMap`, we can use the property `envFrom` on the pod manifest, which is a list that we can use to pass as many environment variables as required.

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

Useful commands:

- `kubectl get configmaps`: list the ConfigMaps inside the default namespace;
- `kubectl describe configmaps`: also list the configuration data of the ConfigMaps.

---

## Secrets

Secrets are similar to ConfigMaps but are specifically intended to hold confidential data.  
Their data are stored in an encoded format by default, but it is desirable to encrypt them.

As with ConfigMaps, there are two phases involved in setting Secrets:

1) create the Secret;
2) inject it into the pod.

In addition, there are also different kinds of Secrets, with different use cases and expected keys, on which some checks can be performed by Kubernetes.

| Secret type                           | Typical use                             | Expected keys           |
| ------------------------------------- | --------------------------------------- | ----------------------- |
| `Opaque`                              | arbitrary user-defined data             | any keys                |
| `kubernetes.io/tls`                   | data for a TLS client or server         | `tls.crt`, `tls.key`    |
| `kubernetes.io/dockerconfigjson`      | serialized `~/.docker/config.json` file | `.dockerconfigjson`     |
| `kubernetes.io/dockercfg`             | serialized `~/.dockercfg` file          | `.dockercfg`            |
| `kubernetes.io/basic-auth`            | credentials for basic authentication    | `username`, `password`  |
| `kubernetes.io/ssh-auth`              | credentials for SSH authentication      | `ssh-privatekey`        |
| `bootstrap.kubernetes.io/token`       | bootstrap token data                    | many keys expected      |
| `kubernetes.io/service-account-token` | `ServiceAccount` token                  | automatically generated |

`Opaque` is the default `Secret` type.  
When you create a `Secret` using kubectl, you must use the `generic` subcommand to indicate an `Opaque` Secret type.

As with ConfigMaps, we can also create Secrets in two ways:

- imperative: `kubectl create secret` specifying the required arguments;
- declarative: `kubectl create -f [file name]`.

Imperatively, this can be done by declaring the key values in the `kubectl` command

```bash
kubectl create secret generic <secret name> --from-literal=<key>=<value>
kubectl create secret generic app-secret --from-literal=DB_Host=mysql --from-literal=DB_User=root --from-literal=DB_Password=paswrd
```

(but this can be complicated when we need to pass many values)

or (still imperatively) through a file (not a manifest)

```bash
kubectl create secret generic <secret name> --from-file=<path-to-file>
kubectl create secret generic app-secret --from-file=app_secret.properties
```

With the declarative approach instead, we do the following

```bash
kubectl create -f <file name>
kubectl create -f secret-data.yaml
```

When using the declarative approach, we should specify the data in `Base64` format.  
For this purpose, we can use the `base64` Linux utility `echo -n 'VALUE' | base64`:

```console
$ echo -n 'mysql' | base64
bXlzcWw=
$ echo -n 'root' | base64
cm9vdA==
$ echo -n 'paswrd' | base64
cGFzd3Jk
```

which makes the secret definition file look like the following

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

We can also prepare the manifest without encoding the data by using the `stringData` field instead of `data`.  
For example, the following example is equivalent of the previous one

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: app-secret
stringData:
  DB_Host: mysql
  DB_User: root
  DB_Password: paswrd
```

> [!CAUTION]
>
> The `data` field stores values encoded in Base64.  
> The `stringData` field accepts plain text values and is automatically converted by the Kubernetes API server to Base64 when the `Secret` is created or updated, and stored into the `data` field.
>
> Base64 is **not encryption**. It is only an encoding format used to represent binary data as text.  
> Kubernetes Secrets are, by default, stored unencrypted in the API server's underlying data store (etcd).  
> In order to safely use Secrets, take at least the following steps:
>
> - enable Encryption at Rest for Secrets;
> - enable or configure RBAC rules with least-privilege access to Secrets;
> - restrict secrets access to specific containers;
> - consider using External Secret store providers (like AWS Provider, Azure Provider, GCP Provider or Vault Provider).

In order to decode the encoded values, use the same `base64` utility adding a `--decode` parameter, so `echo -n 'DATA' | base64 --decode`:

```console
$ echo -n 'bXlzcWw=' | base64 --decode
mysql

$ echo -n 'cm9vdA==' | base64 --decode
root

$ echo -n 'cGFzd3Jk' | base64 --decode
paswrd
```

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

Useful commands:

- `kubectl get secrets`: list the secrets inside the default namespace;
- `kubectl describe secrets`: shows the attributes of the secrets but **hides the values**;
- `kubectl get secret <secret name> -o yaml`: list the attributes of a specific secret, **along with the encoded values**.

---

## Other options to inject ConfigMaps and Secrets

Along with `configMapRef` and `secretRef`, which let you inject all the key-values of a resource as environment variables,  
there are also the `configMapKeyRef` and `secretKeyRef` options for choosing specific keys to import.

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

Instead of exposing ConfigMaps and Secrets as environment variables, they can also be mounted as volumes.  
In this case, each key becomes a file within the mounted volume with:

- the filename corresponding to the key;
- the file content corresponding to the associated value.

The following example mounts both a `ConfigMap` and a `Secret` as Volumes.

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
    [replace with generated tls.crt content]
    -----END CERTIFICATE-----
  tls.key: |
    -----BEGIN PRIVATE KEY-----
    [replace with generated tls.key content]
    -----END PRIVATE KEY-----
```

> [!NOTE]
>
> Here we used `stringData` for the `tls.crt` and `tls.key` elements of the secret, which allows us to write data in a non-encoded format.  
> When applied, Kubernetes automatically encodes the values of `stringData` in Base64 and puts them instead in the `data` field.
>
> We are also using a Secret of type `kubernetes.io/tls` instead of a generic `Opaque` type.

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
> Here we used `subPath` for the `volumeMount` of the configuration.  
> Without `subPath`, the entire volume is mounted at `mountPath`. With `subPath`, only the specified file or directory within the volume is mounted at `mountPath`.  
>
> In this example, we are essentially saying:  
> > From the nginx-config volume, take only the nginx.conf file and mount it here.
>
> ```text
> /etc/nginx/
>         └── nginx.conf
>
> /etc/nginx/tls/
>             ├── tls.crt
>             └── tls.key
> ```
>
> Using `subPath` is particularly useful when we want to mount a single file or directory **without replacing or hiding the other contents of the target directory**.  
> For example, mounting the entire `nginx-config` volume at `/etc/nginx` could hide files that are already present in `/etc/nginx` in the container image. Using `subPath` allows us to mount only `nginx.conf`, leaving the rest of `/etc/nginx` untouched.  
> Therefore, if we want to mount an entire volume and its contents, we can omit `subPath`. If we want to mount only a specific file or directory while preserving the existing contents of the target directory, we can use `subPath`.
>
> A `subPath` mount also does not receive automatic updates when the underlying `ConfigMap` or `Secret` changes.
>
> We are also using `readOnly: true`, which is optional for Secret and ConfigMap volumes (they are read-only by nature), but explicitly specifying it makes the intent clearer.
>
> Notice that the field used to specify the referenced object is not named consistently: `ConfigMap` volumes use `configMap.name`, while `Secret` volumes use `secret.secretName`. Both fields contain the name of the corresponding Kubernetes object.

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
> - the `ConfigMap` stores the NGINX configuration, which can change frequently;
> - the `Secret` stores the TLS certificate and the private key.
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
      path: tls.crt
    - key: tls.key
      path: tls.key
```

> [!CAUTION]
>
> A `Secret` or `ConfigMap` mounted as a `volume` is not stored inside the container image.  
> Kubernetes creates the files dynamically and mounts them into the container's filesystem.
>
> If the `Secret` or `ConfigMap` is updated, the mounted files are automatically updated as well (with a small delay), without restarting the `Pod`, except when using `subPath` (and other particular cases).  
> Instead, when a `Secret` or `ConfigMap` is exposed as environment variable, the `Pod` must be recreated to see the new values.

```text
                ConfigMap / Secret
                        │
        ┌───────────────┴───────────────┐
        │                               │
        ▼                               ▼
Environment Variables             Mounted Volume
        │                               │
        ▼                               ▼
    env / envFrom              volume + volumeMount
        │                               │
        ▼                               ▼
Available at startup            Available as files
        │                               │
        ▼                               ▼
Pod restart required        Files automatically updated
  (after changes)                (except subPath)
```

> [!TIP]
>
> Use a normal `ConfigMap` or `Secret` volume when you want configuration updates to be propagated automatically to a running `Pod`.  
> Use `subPath` when you need to mount a single file to a specific location in the container's filesystem, but remember that updates to the `ConfigMap` or `Secret` will **not** be reflected until the `Pod` is recreated.

---

## Security Contexts

A security context defines privilege and access control settings for a `Pod` or a container. The most common are:

- `runAsUser`: specifies the Linux User ID (UID) used to run the container processes;
- `runAsGroup`: specifies the Linux Primary Group ID (GID) used to run the container processes;
- `runAsNonRoot`: prevents the container from running as the root user (UID 0);
- `fsGroup`: defines the GID assigned to mounted Volumes;
- Linux Capabilities: for example `NET_ADMIN`, `MAC_ADMIN` or `ALL`, which you can add or remove with `capabilities.add` and `capabilities.drop`.

Some of the security context parameters are definable at container level, some at pod level, while others are definable both at container and pod level.  
The following table shows where some of the main security context parameters are definable.

| Setting                    | Pod                | Container          |
| -------------------------- | ------------------ | ------------------ |
| `runAsUser`                | :white_check_mark: | :white_check_mark: |
| `runAsGroup`               | :white_check_mark: | :white_check_mark: |
| `runAsNonRoot`             | :white_check_mark: | :white_check_mark: |
| `fsGroup`                  | :white_check_mark: | :x:                |
| `supplementalGroups`       | :white_check_mark: | :x:                |
| `seLinuxOptions`           | :white_check_mark: | :white_check_mark: |
| `seccompProfile`           | :white_check_mark: | :white_check_mark: |
| `appArmorProfile`          | :white_check_mark: | :white_check_mark: |
| `capabilities`             | :x:                | :white_check_mark: |
| `privileged`               | :x:                | :white_check_mark: |
| `allowPrivilegeEscalation` | :x:                | :white_check_mark: |
| `readOnlyRootFilesystem`   | :x:                | :white_check_mark: |

> [!NOTE]
>
> A Pod-level `securityContext` defines the default security settings for all containers in the `Pod`.  
> A Container-level `securityContext` applies only to that container and overrides the Pod-level setting whenever the same field can be specified at both levels.

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
- print the current user ID, group ID, and the groups it belongs to with `kubectl exec [pod name] -- id`;
- print the current username with `kubectl exec [pod name] -- whoami`.

---

## Resource requirements

When scheduling a `Pod`, the kube-scheduler considers the resource requests of the Pod and the available resources on each node.  
If a node does not have enough available resources, the kube-scheduler will not schedule the Pod onto that node and will instead search for nodes with sufficient resources available.  
If no node has sufficient available resources, the Pod remains in the `Pending` state until suitable resources become available.

We can specify the amount of CPU and memory guaranteed for a container at creation with `resources.requests`.

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
> this means that any container without limits set can consume as many resources as it requires on any node and suffocate other pods or processes running on the node.

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
>              Pod
>               │
>       resources.requests
>               │
>               ▼
> kube-scheduler chooses a node
>               │
>               ▼
>       Pod starts running
>               │
>       resources.limits
>               │
>               ▼
>    kubelet / container runtime
>     enforce CPU and memory limits
> ```

You can also create objects called `LimitRange` at namespace level in order to define default resource requests and limits set for containers (and also PersistentVolumeClaims) when they are not specified inside the manifest.

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

---

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

---

### The default ServiceAccount

A service account named `default` already exists by default in every Kubernetes namespace.  
Whenever a Pod is created, the default service account is automatically mounted into the Pod as a projected volume on the path `/var/run/secrets/kubernetes.io/serviceaccount`.  
You can see that with the command `kubectl describe pod [pod name]`.

```text
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

---

### Using a custom ServiceAccount

If you want to use a different `ServiceAccount`, specify its name in the `serviceAccountName` field.

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

---

### ServiceAccount tokens

A ServiceAccount is mainly used to authenticate to the Kubernetes API.  
The kubelet automatically obtains the token and projects it into the Pod.

Inside the mounted `ServiceAccount` volume, the `token` file contains the JWT.

```console
$ kubectl exec -it nginx -- cat /var/run/secrets/kubernetes.io/serviceaccount/token
eyJhbGciOiJSUzI1NiIsImtpZCI6ImRSR3pRcTRSamFxTGdqS0hSZHhmN2w4RmtTMTFtZzdhUDJWSlB1NjNrVTgifQ.eyJhdWQiOlsiaHR0cHM6Ly9rdWJlcm5ldGVzLmRlZmF1bHQuc3ZjLmNsdXN0ZXIubG9jYWwiXSwiZXhwIjoxODE1MTQ1OTcyLCJpYXQiOjE3ODM2MDk5NzIsImlzcyI6Imh0dHBzOi8va3ViZXJuZXRlcy5kZWZhdWx0LnN2Yy5jbHVzdGVyLmxvY2FsIiwianRpIjoiYmRhOTNhNDItYjc5ZC00ZDU4LWFiMmQtMGZmYTdmN2I0ZThiIiwia3ViZXJuZXRlcy5pbyI6eyJuYW1lc3BhY2UiOiJkZWZhdWx0Iiwibm9kZSI6eyJuYW1lIjoibWluaWt1YmUiLCJ1aWQiOiIyNmEyOTM0MC0yN2VmLTQ0YzYtYjU0My04YmFlYWVjN2Y0Y2IifSwicG9kIjp7Im5hbWUiOiJuZ2lueCIsInVpZCI6IjVjNzExZDNhLTgxYzMtNDRhZS1hMWUxLTk5OGY1MzM0NmE5ZiJ9LCJzZXJ2aWNlYWNjb3VudCI6eyJuYW1lIjoiZGVmYXVsdCIsInVpZCI6IjE3ZWY0MDhiLTc4YTEtNGU1NC1iMDY0LWEzYzc1NTgyNjA1MiJ9LCJ3YXJuYWZ0ZXIiOjE3ODM2MTM1Nzl9LCJuYmYiOjE3ODM2MDk5NzIsInN1YiI6InN5c3RlbTpzZXJ2aWNlYWNjb3VudDpkZWZhdWx0OmRlZmF1bHQifQ.x_hvNFm86YWoZ29xXFinGhz7tzzZ83IflzQLZdu0TZMMoorFt9IgJGzMEcFnkuDecGYem8gSE-FAmhGss5JUq_5EE7dSrBsc-VF8sqYbGkBq6sSJDxmWkIC22qGcbGC0GcvdP0fffl98BZYs7Twc8B6KoGuO0KqcuwZH4bYsM31T19Gv1ubismH4-MqxM3d_mMhSCBQht_iymi8Bo6zLvGEPhHRgZzTH0u6nRF9BRTFsbeA7WLCcbuo-yAikEGjzbxS5w0LAEkROHc_Piv8ukfUMLrOf0V_66rOXDmE6EQdOHXna8wq37U2NuqEBO_AzUQnXrgJbw7NfcRK7iDweHg
```

You can use the JWT as a Bearer token when authenticating to the Kubernetes REST API (for example with curl, you can pass it in the `Authorization` header).

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

If a long-lived token is still required (similar to the legacy behavior, see later), we can still do that via a `Secret` of type `kubernetes.io/service-account-token`, with the name of the `ServiceAccount` specified in the `metadata.annotations` section.

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
> According to the Kubernetes documentation, a service-account-token Secret object should only be created if
>
> - we cannot use the `TokenRequest` API to obtain a token and
> - the security exposure of persisting a non-expiring token credential is acceptable.

---

### A preview about RBAC, Roles, RoleBindings

If you try to check if the token has the right to query the Kubernetes API with the command

```bash
kubectl auth can-i get pods --as=system:serviceaccount:default:dashboard-sa
```

you will receive a `no` as a response, because no permissions have been assigned to it.

If, instead, you create a `Role` and assign it with a `RoleBinding`

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
  kind: Role # must be Role or ClusterRole
  name: pod-reader # must match the name of the Role or ClusterRole you wish to bind to
  apiGroup: rbac.authorization.k8s.io
```

```bash
kubectl apply -f pod-reader-role.yaml
kubectl apply -f dashboard-sa-role-binding.yaml
```

If you re-execute the command above, you will see that the response is now `yes`, because the token has sufficient rights.

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

This topic will be covered in more detail later.

---

### Manually mount a ServiceAccount token

You manually mount a ServiceAccount token by disabling automatic mounting of API credentials and using a projected volume.

TODO: Complete this section.
      Check the last part of the lab on ServiceAccount.

---

### Evolution of ServiceAccount tokens

Service account tokens have evolved over time.

Prior to Kubernetes v1.24, creating a ServiceAccount automatically created a Secret containing a long-lived token.  
The Secret name appeared in the `tokens` field of the ServiceAccount.  
In that case, you can directly mount the Secret (containing the token) as a volume of a Pod (instead of exporting it and then injecting it in some way).

In Kubernetes v1.21 and earlier, that default token was not time bound (so it did not have an expiration and was valid as long as the Service Account existed) and was not audience bound.  
In version 1.22 the TokenRequest API was introduced as part of the enhancement proposal KEP-1205, which introduced a more secure and scalable mechanism for provisioning ServiceAccount tokens.  
Tokens generated through this API are:

- time bound
- audience bound
- object bound

This makes `ServiceAccountTokens` significantly more secure.  
Instead of mounting a `Secret` containing a long-lived token, Kubernetes now obtains a short-lived token through the TokenRequest API and projects it directly into the Pod.

Kubernetes v1.24 introduced KEP-2799: "Reduction of Secret-based Service Account Tokens".  
Here, when creating a ServiceAccount, the Secret is not automatically created.  
You must run the command `kubectl create token dashboard-sa` to generate a token for that ServiceAccount; the token will be printed to the screen when you run this command.

| Version | Behavior                                                   |
| ------- | ---------------------------------------------------------- |
| ≤1.21   | Secret containing a long-lived token automatically created |
| 1.22    | TokenRequest API introduced                                |
| ≥1.24   | Secret no longer automatically created                     |

---

## Taints and Tolerations

Taints and Tolerations define the relationship between Pods and nodes.

- Taints are set on nodes and act as a repellent to Pods;
- Tolerations are set on pods and allow (but do not force) them to be scheduled onto nodes with matching taints.

By default, Pods have no tolerations and this means that Pods cannot be scheduled onto tainted nodes unless they define a matching toleration.  
In other words, a pod with no toleration will never be run on a node with a taint.

To define a taint on a node, run the following command

```bash
kubectl taint nodes node-name [key]=[value]:[taint-effect]
```

The taint effects are:

- `NoSchedule`: the pod will not be scheduled on the node;
- `PreferNoSchedule`: the system will try to avoid placing a pod on a node (but is not guaranteed);
- `NoExecute`: new pods will not be scheduled on the node, and any existing pods that do not tolerate the taint will be evicted.

For example

```bash
kubectl taint nodes node1 key1=value1:NoSchedule
```

places a taint on node `node1`. The taint has key `key1`, value `value1`, and taint effect `NoSchedule`. This means that no Pod will be scheduled onto `node1` unless it has a matching toleration.

> [!TIP]
>
> Control plane nodes, even though they have the capability to run pods, are not taken into account by the scheduler because, when the cluster is first set up, a taint is automatically placed on them.
>
> ```console
> $ kubectl describe node controlplane | grep Taint
> Taints:             node-role.kubernetes.io/control-plane:NoSchedule
> ```
>
> This behavior can be modified as required but a best practice is to not deploy applications on the master nodes.

You can then specify a toleration for a pod in the PodSpec.  
Both of the following tolerations "match" the taint created by the `kubectl taint` line above, and thus a pod with either toleration would be scheduled onto `node1`:

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
If this happens and the node also has a `NoExecute` taint set, the kubelet will eject the Pod unless there is an appropriate toleration set.

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
>
> There are two special cases:
>
> If the `key` is empty, then the `operator` must be `Exists`, which matches all keys and values. Note that the `effect` still needs to be matched at the same time.
>
> An empty `effect` matches all effects for that key.

---

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
          - key: zone
            operator: In
            values:
            - EU
```

There are currently two types of node affinity:

- `requiredDuringSchedulingIgnoredDuringExecution`: the scheduler can't schedule the Pod unless the rule is met. This works like `nodeSelector`, but with a more expressive syntax;
- `preferredDuringSchedulingIgnoredDuringExecution`: the scheduler tries to find a node that meets the rule. If a matching node is not available, the scheduler still schedules the Pod.

> [!NOTE]
>
> In the preceding types, `IgnoredDuringExecution` means that if the node labels change after Kubernetes schedules the Pod, the Pod continues to run.

Inside `nodeSelectorTerms[*].matchExpressions` you define the key-value pairs in the form key - operator - value.

The types of operators are:

- `In`: The label value is present in the supplied set of strings;
- `NotIn`: The label value is not contained in the supplied set of strings;
- `Exists`: A label with this key exists on the object;
- `DoesNotExist`: No label with this key exists on the object.

If you want to also allow the pod to be placed on a node labeled with medium, you can just add it to the `values` list

```yaml
        - matchExpressions:
          - key: size
            operator: In
            values:
            - Large
            - Medium
```

If, instead, you want to prevent the pod from being placed on a node labeled with small, you can use the following syntax

```yaml
        - matchExpressions:
          - key: size
            operator: NotIn
            values:
            - Small
```

And now for an example using the `Exists` operator: Medium and Large nodes are labeled while small nodes aren't.  
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

---

## Multi-Container Pods and InitContainers

A Pod can contain more than one container.  
All containers in the same Pod:

- share the same network namespace;
- share the same IP address;
- can share volumes.

Regular (co-located) containers in a Pod also share the same lifecycle; init containers and sidecar containers, as we'll see below, can have a different lifecycle.

There are different patterns of multi-container pods:

- co-located containers: the services are dependent on each other and share their entire lifecycle inside the pod;
- init containers: there are initialization steps to be performed when the pod starts (for example waiting for a DB to be ready or pulling code or a binary from a repository that will be used by the main web application);
- sidecar container: the sidecar starts first, does its job, and continues to run alongside the main application container, terminating only after the main container stops (for example a log shipper).

With co-located containers, there is no guarantee that one container starts before another.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: simple-webapp
  labels:
    name: simple-webapp
spec:
  containers:
  - name: web-app
    image: web-app
    ports:
    - containerPort: 8080
  - name: main-app
    image: main-app
```

Using the `initContainers` field, you can also define more than one init container; in that case each init container runs one at a time, in sequential order.  
In the following example, `db-checker` runs first and then ends; then `api-checker` runs and ends; finally, the main application runs.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: simple-webapp
  labels:
    name: simple-webapp
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

If any of the init containers fail to complete, Kubernetes restarts that specific init container repeatedly until it succeeds, before moving on to the next one.

In order to achieve the sidecar container pattern, you need to add `restartPolicy: Always` to the specific init container's own definition (not to the Pod's `spec.restartPolicy`, which is a different setting). This ensures the init container will continue to run and will be terminated only after the main application stops (in this way the log shipper can catch the startup and termination logs of the main container).

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: simple-webapp
  labels:
    name: simple-webapp
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

---

## Observability

---

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
- `Succeeded`: all containers in the Pod have terminated successfully, and will not be restarted;
- `Failed`: all containers in the Pod have terminated, and at least one container has terminated in failure. That is, the container either exited with non-zero status or was terminated by the system, and is not set for automatic restarting;
- `Unknown`: for some reason the state of the Pod could not be obtained. This usually indicates a communication problem between the control plane and the node where the Pod should be running.

> [!TIP]
>
> The `STATUS` column displayed by `kubectl get pods` does not always correspond to the Pod phase and it's instead a "user-friendly" representation built by kubectl.
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

The Pod's status includes an array of `PodConditions` (of true or false values) that indicate whether the Pod has passed certain checkpoints.  
As a Pod progresses through its lifecycle, the Kubelet sets the following conditions roughly in this order:

- `PodScheduled`: the pod has been scheduled on a node;
- `PodReadyToStartContainers`: the Pod sandbox has been successfully created and networking configured. The sandbox and network are set up by the container runtime and CNI plugin;
- `Initialized`: all `init containers` have completed successfully (for a Pod without init containers, this is set to True before sandbox creation);
- `ContainersReady`: all containers in the pod are ready;
- `Ready`: the Pod is able to serve requests and should be added to the load balancing pools of all matching Services.

Not every Pod will necessarily expose all conditions, depending on the Kubernetes version and enabled features.

When running `kubectl describe pod <pod name>`, the Pod Conditions appear like this:

```text
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

---

### Readiness and Liveness Probes

As a developer of the application, you know better what it means for the application to be ready.

There are different ways to test / probe if an application inside a container is actually ready by adding a `readinessProbe` in the Pod manifest.  
Some of them are:

- calling an API over HTTP, in the case of a web application;

  ```yaml
      readinessProbe:
        httpGet:
          path: /api/ready
          port: 8080
  ```

- checking that a particular TCP socket is listening, for a database;

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

Until the `readinessProbe` completes successfully, the Service does not forward any traffic to the Pod.

There are other options, like:

- add an additional delay to the probe (for example if you know that the application takes a minimum amount of time to warm up);
- specify how often to probe with the `periodSeconds` option;
- specify a different threshold of consecutive failures before Kubernetes considers the probe failed (three by default, if not specified).

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: simple-webapp
  labels:
    name: simple-webapp
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
For example, when you have a ReplicaSet with multiple pods, the Service routes traffic only to the ready pods.

A Liveness probe can be configured on a container to periodically test whether the application in the container is healthy.  
If the test fails, the container is considered unhealthy and it will be destroyed and recreated.

Again, as a developer of the application, you must define what it means for the application to be healthy.

The Liveness Probe is defined in the Pod definition file in a similar way to what we already did with the readiness probe.

```yaml
      livenessProbe:
        exec:
          command:
          - sh
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

---

### Monitoring, Logging, and Debugging

Kubernetes provides several commands to inspect logs, monitor resource usage and troubleshoot applications.

For the logs we can use `kubectl logs`.  
By default, the logs are retrieved from the main container but we can select a specific container using the `--container` or `-c` option.  
We can also retrieve the logs from all the containers of a pod with the option `--all-containers`.

There are also options for retrieving useful metrics via the `Metrics Server`, an add-on component that is not installed by default.  
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

---

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

In the example above, we added the following label keys and values on a pod

| Key      | Value     |
| -------- | --------- |
| app      | app1      |
| function | front-end |

You can then use selectors to filter the output of kubectl commands based on labels.

```bash
kubectl get pods --selector app=app1
```

You can also filter using multiple labels by separating them with commas.  
The short form of `--selector` is `-l`.

```bash
kubectl get pod -l env=prod,bu=finance,tier=frontend
```

Kubernetes objects use labels and selectors internally to connect different objects together.  
For example, the following command for creating a deployment

```bash
kubectl create deployment nginx --image=nginx --replicas=3
```

automatically creates:

- a label (`app: nginx`) on the `Deployment`;
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

The `Deployment` manages Pods by selecting them through the labels defined in the Pod template.

> [!TIP]
>
> Although a single label is often sufficient, using multiple labels generally makes selectors more precise and expressive.  
> For example, selecting only `app=web` may match frontend Pods, backend Pods, Jobs or CronJobs.  
> Combining multiple labels such as `app=web,tier=frontend,env=prod` reduces the chance of selecting unintended objects.
>
> You can also use `matchExpressions` instead of `matchLabels` to have more granular control over selectors.

When a Service is created, it uses its selector to match the labels of the target Pods.  
In many cases those Pods are created by a `ReplicaSet` or `Deployment`, but Services always select Pods directly.

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
    buildVersion: "1.34"
    imageregistry: "https://docker.com"
    description: "Main frontend Nginx web server"
spec:
  containers:
  - name: nginx-container
    image: nginx:1.21.6
```

---

## Deployment Updates and Rollback

When you first create a `Deployment`, it triggers a rollout.  
A new rollout creates a new `ReplicaSet`, which is recorded as a new `Deployment` revision.  
Whenever the Pod template (`spec.template`) changes (for example, the container image, environment variables, commands and arguments, volumes, resource limits, labels, annotations, etc), a new rollout is triggered and a new `Deployment` revision is created.  
This helps us keep track of the changes made to our deployment, enabling rollback capabilities if necessary.

> [!IMPORTANT]
>
> Only changes to the Pod template (`spec.template`) create a new `ReplicaSet`.  
> Scaling a Deployment (`spec.replicas`) updates the existing `ReplicaSet` but does **not** create a new `ReplicaSet` or a new `Deployment` revision.

You can see the status of a rollout with the command `kubectl rollout status deployment/myapp`.  
To see the rollout history and the available revisions, use `kubectl rollout history Deployment myapp`.  
To see the details of a specific revision, use the command `kubectl rollout history deployment/myapp --revision=2`.

> [!NOTE]
>
> You can use interchangeably `deployment/deployment_name`, `Deployment deployment_name`, `deployment deployment_name`, `deploy deployment_name`

In Kubernetes, there are two types of deployment strategies:

- `Recreate`: scales the old `ReplicaSet` down to zero before scaling the new `ReplicaSet` up to the desired amount, causing downtime during the update;
- `RollingUpdate`: the old `ReplicaSet` is gradually scaled down while the new `ReplicaSet` is scaled up, progressively replacing old Pods with new ones until all Pods have been replaced, ensuring that part of the application remains available during the update.

The Deployment strategy is configured under `spec.strategy`, and `strategy.type` determines how Pods are replaced during a Deployment update; the default deployment strategy is `RollingUpdate`.

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
For example, if a Deployment has 8 replicas and `maxSurge: 25%`, Kubernetes may temporarily run 8 old Pods + 2 new Pods = 10 Pods.

`maxUnavailable`, instead, defines how many Pods are allowed to be unavailable during the update.  
For example, if a Deployment has 8 replicas and `maxUnavailable: 25%`, Kubernetes may temporarily have 6 available Pods and 2 unavailable Pods.

Both `maxSurge` and `maxUnavailable` can be specified as an absolute number or a percentage, and their default value is 25%.

As an example, consider a RollingUpdate with the following configuration:

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

Kubernetes Deployments allow us to roll back a Deployment to a previous revision, with the `kubectl rollout undo` command, for example:

- `kubectl rollout undo deployment/myapp`: if you want to roll back to the previous Deployment revision;
- `kubectl rollout undo Deployment myapp --to-revision=2`: if you want to roll back to a specific revision.

The `Deployment` then removes the Pods belonging to the current `ReplicaSet` and scales the previous `ReplicaSet` back up.

```text
kubectl create deployment
Deployment --► Revision 1 --► ReplicaSet #1 --► Pods v1

kubectl set image / kubectl apply / kubectl edit
Deployment --► Revision 2 --► ReplicaSet #2 --► Pods v2

kubectl rollout undo
ReplicaSet #1 becomes active again and ReplicaSet #2 is scaled down
```

Finally, if you need to restart the pods of the current revision of a deployment (like to force a Pod restart after an update of a ConfigMap or a Secret exposed through environment variables), you can use the command `kubectl rollout restart deployment/myapp`.  
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

  in the output you can notice interesting fields like `Replicas`, `StrategyType`, `RollingUpdateStrategy`, `OldReplicaSets` and `NewReplicaSet`.

- **Using the `--revision` flag**
  
  Here, revision 1 is the first version of the deployment.  
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

  Let's now roll back to the previous revision:

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
  
  With this, we have rolled back to the previous version of the deployment with the image `nginx:1.29`.

  ```console
  $ kubectl describe deployments nginx | grep -i image:
      Image:        nginx:1.29
  ```
  
  To roll back to a specific revision, we will use the `--to-revision` flag.  
  With `--to-revision=1`, it will be rolled back to the first image we used to create the deployment, as we can see in the rollout history output.

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
      Image:        nginx:1.30
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

---

### Additional Deployment strategies

Besides the built-in Deployment strategies (`RollingUpdate` and `Recreate`), there are other application deployment strategies commonly used in production, such as:

- Blue-Green deployment;
- Canary deployment.

These are not configurable in the Kubernetes Deployment manifest, but they can be implemented in other ways.  
They are best implemented with service meshes like Istio, but let's see how to implement them with Kubernetes primitives.

The **Blue-Green deployment** procedure is as follows:

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
5) change the selector on the service to `version: v2`.

    ```yaml
    kind: Service
    spec:
      selector:
        version: v2
    ```

The **Canary deployment** procedure is as follows:

1) the old version is deployed at full desired replicas;
2) all the traffic is routed to the old version (primary);
3) the new version is deployed with few replicas (canary);
4) we start to route only a small portion of traffic to the canary deployment;
5) tests are performed on the new version, which is partly live with its few replicas;
6) once all tests are passed, we upgrade the primary deployment with the new version of the application (maybe making use of rollingUpdate strategy);
7) we delete the canary deployment.

and we can implement it in K8s in this way:

1) create the first deployment called `primary` with a full amount of replicas (for example 5 pods) with the labels `version: v1` and `app: front-end`;

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

3) create the second deployment called `canary` with the label `version: v2` and a reduced amount of replicas (for example 1 pod) with the labels `version: v2` and `app: front-end`;

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

4) change the Service selector to `app: front-end`, so that both Deployments become Endpoints of the same Service, with a smaller percentage on the canary one;

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

One of the caveats of the canary deployment is that we have limited control over the split of traffic through the deployments.  
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

---

## Jobs and CronJobs

Kubernetes supports different types of workloads depending on the application lifecycle, which can be broadly divided into long-running workloads and batch workloads.  
For long-running applications (such as web applications), Deployments are typically used.  
For jobs like batch processing, analytics or reporting, that are meant to live for a short period of time, perform specific tasks, and then finish, the `Job` is a good choice.

> [!CAUTION]
>
> If you create a Pod instead of a Job to perform a short-lived task, like in this example
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
> you'll risk triggering multiple restarts of the container because after it completes, Kubernetes continues to bring the container up again.  
> This is because the default restart policy for Pods is `Always`, which involves restarting the container in an attempt to keep it running, until a threshold is reached.  
> This behavior is defined by the property `spec.restartPolicy`, which by default is set to `Always`.  
> You can override this behavior by setting this property to `Never` or `OnFailure`.  
> In any case, this pattern is discouraged, and short-lived tasks should be defined as Jobs or CronJobs instead of Pods.

While `ReplicaSet` is used to make sure a specified number of pods is running at all times, a `Job` is used to run a set of pods to perform a given task to completion.

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

We can also see the standard output of a Job's containers with the command `kubectl logs <pod name>` or `kubectl logs job/<job name>`

```console
$ kubectl get pods
NAME                     READY   STATUS      RESTARTS        AGE
math-add-job-xxxbx       0/1     Completed   0               4m39s

$ kubectl logs math-add-job-xxxbx
5
```

Finally, to delete the job, run `kubectl delete job <job-name>`

```console
$ kubectl delete job math-add-job
job.batch "math-add-job" deleted
```

deleting the Job also deletes the Pods it created.

```console
$ kubectl get pods
No resources found in default namespace.
```

If the Job must complete successfully multiple times, set the `.spec.completions` parameter.

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

If `.spec.completions` is set to 3, Kubernetes waits until three Pods complete successfully.  
Failed Pods do not count towards the completion count.  
If the pods fail (for example because the application exits with an error), the Job counts only successful Pod completions toward `.spec.completions` while failed Pods are retried until `.spec.backoffLimit` is reached.

There are situations where you want a Job to fail after a certain number of retries, for example due to a logical error in the configuration.  
To do so, set `.spec.backoffLimit` to specify the number of retries before considering a Job as failed.  
Every failed Pod counts towards the `backoffLimit`. Once this limit is reached, the Job is marked as failed.  
The `.spec.backoffLimit` is set by default to 6.  
In this case (where the Pod randomly fails) we want to increase it, so that the Job still reaches 3 successful completions despite the random failures.

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
random-error-job-kzl9g   0/1     Completed   0          61s
random-error-job-ljwrd   0/1     Completed   0          23s
random-error-job-q7xkl   0/1     Completed   0          27s
random-error-job-xgbs5   0/1     Error       0          60s
```

By default the pods are created one after the other (the next pod is created only after the previous is finished).  
We can have the Pods created in parallel instead of sequentially by setting the `spec.parallelism` property.

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

In this example, the Job first creates 3 pods at once, and then continues in rounds, each time creating a number of pods equal to the number of completions still missing.

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

Another common optional parameter for Jobs is `.spec.activeDeadlineSeconds`, which applies to the duration of the job, no matter how many Pods are created. Once a Job reaches this threshold, all of its running Pods are terminated and the Job's status condition becomes `Failed`, with reason `DeadlineExceeded`.

> [!NOTE]
>
> `parallelism` controls how many Pods may run at the same time.  
> `completions` controls how many successful Pod executions are required before the Job finishes.
>
> `spec.completions` and `spec.parallelism`, if omitted, are set to their default value, which is 1.

A CronJob is a resource that creates Jobs on a repeating schedule.

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

We can notice that the definition of a CronJob is more complex than the one of a Job: there are now three `spec` sections: one for the CronJob, one for the Job and one for the Pod.  
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
- `.spec.jobTemplate.spec.ttlSecondsAfterFinished`: it specifies how long Kubernetes keeps a completed Job before automatically deleting it (it will delete the Job cascadingly, i.e. delete its dependent objects, such as Pods, together with the Job). Note that this field belongs to the Job spec, so within a CronJob it must be set inside `jobTemplate.spec`, not directly under the CronJob's own `spec`.

| Resource   | Long running       | Executes once      | Scheduled          |
| ---------- | ------------------ | ------------------ | ------------------ |
| Pod        | :white_check_mark: | :x:                | :x:                |
| Deployment | :white_check_mark: | :x:                | :x:                |
| Job        | :x:                | :white_check_mark: | :x:                |
| CronJob    | :x:                | :white_check_mark: | :white_check_mark: |

---

## Network Policies

---

### Description of Ingress and Egress in non-Kubernetes environments

Network traffic can be viewed from two perspectives: **ingress** (incoming traffic) and **egress** (outgoing traffic).

For example, in a web application that:

- receives client requests on the web server on port 80;
- sends a request to the app server exposing an API on port 5000;
- interacts with the database on port 3306.

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
  - the outgoing request to the app server on port 5000 is egress traffic;

- for the app that serves API:

  - the incoming traffic from the web server through port 5000 is ingress;
  - the outgoing request to the database on port 3306 is egress traffic;

- for the database:

  - it receives ingress traffic on port 3306 from the API server.

If we were to list the necessary rules to make this system work, we would have:

- an ingress rule that accepts HTTP traffic on port 80 on the web server;
- an egress rule to allow traffic from the web server to port 5000 of the API server;
- an ingress rule to accept traffic on port 5000 on the API server;
- an egress rule to allow traffic to port 3306 on the database server;
- an ingress rule on the database server to accept traffic on port 3306.

---

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

However, since by default all pods can talk to each other, the front-end pod would also be able to communicate directly with the database pod.  
If we want to avoid that (maybe the security team or an audit requires preventing this from happening), that is where we would implement a **Network Policy** to allow traffic to the DB server only from the API server.

---

### Network Policies in Kubernetes environments

A `NetworkPolicy` is a namespaced Kubernetes resource that defines which network connections are allowed to and/or from a selected set of Pods.  
In order to apply a `NetworkPolicy` on a pod we use labels and selectors: we label the pod and we use the same label on the `podSelector` field in the `NetworkPolicy`.  
When building the rule, under `spec.policyTypes` we can specify whether the NetworkPolicy controls Ingress traffic, Egress traffic, or both.  
Finally, we define the allowed sources or destinations and, optionally, restrict them to specific ports and protocols.  
Once a Pod is selected by an Ingress `NetworkPolicy`, all ingress traffic is denied unless explicitly allowed by one or more matching `NetworkPolicies`.

> [!IMPORTANT]
>
> NetworkPolicies are additive.  
> If multiple NetworkPolicies select the same Pod, the allowed traffic is the union of all rules.
>
> Kubernetes does not evaluate NetworkPolicies in order.

In the case above, we would say "only allow ingress traffic from the API pod on port 3306".  

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
- Cilium
- Weave Net

If we use Flannel for example, it does not currently support NetworkPolicies.  
In any case, even in a cluster configured with a networking solution that does not support NetworkPolicies, we can still create the policies, but they will simply not be enforced.  
In other words, NetworkPolicies are defined by Kubernetes, but enforcement is delegated to the CNI plugin.

Thinking about ingress and egress traffic, you could think that:

- the outgoing response of an incoming request;
- the incoming response of an outgoing request;

needs a different rule; however no additional rule is needed because NetworkPolicies are stateful: once a connection is allowed, the corresponding return traffic is automatically permitted.  
So when defining NetworkPolicies, you don't need to worry about response traffic.

There are additional selectors that can be used in `NetworkPolicies`, like

- `namespaceSelector`: defines from which namespace the traffic is allowed to reach the pod.

  ```yaml
      - namespaceSelector:
          matchLabels:
            kubernetes.io/metadata.name: prod
  ```

- `ipBlock`: defines a range of IP addresses from which to allow traffic to the pod. It's commonly used to allow traffic from external IP addresses or external networks.

  ```yaml
      - ipBlock:
          cidr: 192.168.5.10/32
  ```

The same applies to egress traffic.

| Selector          | Matches    |
| ----------------- | ---------- |
| podSelector       | Pods       |
| namespaceSelector | Namespaces |
| ipBlock           | IP ranges  |

Those rule selectors can be passed separately (as separate rules) or grouped together (as part of a single rule).

> [!IMPORTANT]
>
> Inside a single `from` (or `to`) entry, different selectors are combined with a logical AND.  
> Multiple entries in the `from` (or `to`) list are combined with a logical OR.

In the following example, we have two ingress elements / rules:

- the first rule states that traffic from pods labeled with `name: api-pod` in the `prod` namespace is allowed;
- the second rule states that traffic from the IP range `192.168.5.10/32` is allowed;

Traffic from sources meeting either of these criteria is allowed to pass through.

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
  name: default-deny-all
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

---

## Ingress Networking

---

### Why do we need Ingress instead of exposing Services directly?

With a Service, if we want to expose an application to the external world, we can use a Service of type NodePort.  
NodePort exposes the Service on a high TCP port (typically in the 30000–32767 range). Although functional, this is generally not suitable for production because users must access non-standard ports and additional infrastructure is usually required to provide a stable HTTP/HTTPS entry point.
To remedy this, we could add a reverse proxy that accepts requests on port 80 and redirects them to the port exposed by the NodePort.  
The reverse proxy provides the public entry point, handles HTTP/HTTPS traffic and forwards requests to the NodePort Service.  
We can then point our DNS to this server and finally users are able to access the application by simply visiting the FQDN.  
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
external infra   |
════════════════════════════════════
      Kubernetes | Cluster
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
cloud provider   │
════════════════════════════════════
      Kubernetes | Cluster
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
- in case of cloud provider: create a new Service of type `LoadBalancer` which provisions another Load Balancer component set on a different port and with a different IP, which will result in increased costs.

If both applications must be exposed under the same FQDN, something must inspect the incoming URL and decide where to send the request.  
This requires an additional reverse proxy (or another load balancer) capable of URL-based routing.  
Every time a new application is deployed, this external component must be reconfigured.  
This means that routing configuration lives outside Kubernetes, making deployments harder to automate and maintain.

> [!TIP]
>
> The routing may also be based on the requested host (for example shop.example.com and api.example.com) instead of the URL path.

Finally, we also need to enable SSL for the applications, so the users can access the application using HTTPS.  
We can do this at different levels:

- application level;
- load balancer / proxy-server level.

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
          │                        │
  NodeIP:30080               NodeIP:30082
          │   cloud provider       │
════════════════════════════════════════════════
          |   Kubernetes Cluster   |
 +----------------+       +----------------+
 | Service 1 (LB) |       | Service 2 (LB) |
 |     :30080     |       |     :30082     |
 +----------------+       +----------------+
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
═══════════════════════════════════════════
Kubernetes Cluster   |
          +--------------------+
          | Service (NP or LB) |
          +--------------------+
                     │
   +-----------------------------------+
   |        Ingress Controller         |
   +-----------------------------------+
    /app1 |                  /app2 |
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

---

### Ingress and Ingress Controller

Think of Ingress as a Layer 7 load balancer built into the Kubernetes cluster that can be configured using native Kubernetes primitives.

In order to implement Ingresses, you need two elements

- **Ingress Controller**: deploy a reverse proxy or a load balancing solution on the Kubernetes Cluster;
- **Ingress resources**: define the configuration (routes, certificates, etc) in `Ingress` Kubernetes manifests.

> [!NOTE]
>
> If you simply create Ingress resources (without deploying an Ingress controller) they won't work.  
> A Kubernetes cluster does not come with an Ingress Controller by default.

There are many available solutions for Ingress controller like NGINX, HAProxy, Traefik, GCE (Google Layer 7 Load Balancer), Contour.  
Out of these, NGINX and GCE are currently being supported and maintained by the Kubernetes project.

Keep in mind that the load balancer component is just a part of Ingress Controllers.  
Ingress Controllers have additional intelligence to monitor the Kubernetes Cluster for definitions of ingress resources and configure the underlying proxy accordingly.

---

### Simplified architecture of an Ingress Controller

> [!NOTE]
>
> Modern Kubernetes clusters usually install Ingress Controllers using Helm charts or vendor-provided manifests.  
> The following Deployment is intentionally simplified to illustrate the components involved.

In the following example we'll use NGINX as an Ingress Controller.  
The NGINX controller is deployed just like any other Deployment in Kubernetes.

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
        name: nginx-ingress
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
The NGINX controller needs them to locate and read the ConfigMap associated with the Pod.

Finally, we specify the ports used by the Ingress Controller, which are `80` and `443`.

We also add an empty ConfigMap, which can later be used to pass configuration values (like `err-log-path`, `keep-alive` and `ssl-protocols`)

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

As said before, the Ingress Controller can monitor the cluster and configure the underlying NGINX server when something changes.  
But in order for the Ingress Controller to do this, we need a `ServiceAccount` with the right set of permissions defined with `Roles`, `ClusterRoles` and `RoleBindings`.

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

---

### Ingress resources

Let's now define some Ingress rules on the Ingress Controller via `Ingress` resources.

You can create Ingress rules to, for example:

- simply forward Ingress traffic to a single application;
- route traffic to different applications based on URLs;
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
NAME           CLASS   HOSTS   ADDRESS   PORTS   AGE
ingress-wear   nginx   *                 80      32s
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
Ingress Class:    nginx
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

You can also notice that the `Default backend` field shows `<default>`, indicating that none has been explicitly configured.  
We can specify the `defaultBackend`, which defines the backend that receives traffic that does not match any of the rules.  
If omitted, unmatched requests typically receive a 404 response generated by the Ingress Controller.

The third configuration option uses domain names or host names.  
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

You can also add TLS to an Ingress

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

---

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

The corresponding Ingress then becomes

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

---

### Additional options

Different Ingress Controllers have different options that can be used to customize how they work.

For example, [NGINX Ingress Controller has many options](https://kubernetes.github.io/ingress-nginx/examples/) that can be passed as environment variables or annotations.

One useful argument to pass to the NGINX Ingress Controller allows the controller to route unmatched requests to a default `Service`, even if no default backend is defined in the `Ingress` resource.

```yaml
        args:
        - --default-backend-service=<namespace>/<service>
```

Some common examples of annotations for the NGINX Ingress Controller are:

| Annotation                                    | Purpose                                                               |
| --------------------------------------------- | --------------------------------------------------------------------- |
| `nginx.ingress.kubernetes.io/rewrite-target`  | Rewrites the request path before forwarding it to the backend Service |
| `nginx.ingress.kubernetes.io/ssl-redirect`    | Redirects HTTP requests to HTTPS                                      |
| `nginx.ingress.kubernetes.io/proxy-body-size` | Sets the maximum accepted request body size                           |

These annotations are specific to the NGINX Ingress Controller. Other Ingress Controllers may use different annotations or configuration mechanisms.

Useful commands:

- `kubectl create ingress <ingress-name> --rule="host/path=service:port"`: template for creating an Ingress imperatively;
- `kubectl create ingress ingress-test --rule="wear.my-online-store.com/wear*=wear-service:80"`: creates an Ingress that routes requests for `wear.my-online-store.com` whose path starts with `/wear` to the `wear-service` Service on port 80.

---

## State Persistence: Storage, Volumes and StatefulSets

---

### Storage in Docker

---

#### How Docker stores data

When working with Docker, two related concepts come up: **Storage Drivers** and **Volume Drivers**.

By default, Docker stores its local data under `/var/lib/docker` with the following structure:

```text
/var/lib/docker
          ├── overlay2 (storage driver data: image and container layers)
          ├── containers (data related to containers)
          ├── image (data related to images)
          └── volumes (volumes data)
```

---

#### Docker layered filesystem

Docker images use a layered filesystem: most of the instructions in a Dockerfile create a new immutable image layer containing only the changes introduced by that instruction.  
When a container is started, Docker adds a thin writable layer on top of the image layers. All changes made by the running container (new files, modified files and deleted files) are stored in this writable layer.

The writable container layer exists only for the lifetime of the container; when the container is destroyed, this layer and all its changes are destroyed too, while the image layers are still intact (and possibly are also shared with other images).

```text
                 Docker Image
      +-------------------------------+
      | Layer 3 (read-only)           |
      +-------------------------------+
      | Layer 2 (read-only)           |
      +-------------------------------+
      | Layer 1 (read-only)           |
      +-------------------------------+
                   ▲
                   │
        Writable Container Layer
      +-------------------------------+
      | runtime changes               |
      +-------------------------------+
```

---

#### Why we need volumes in Docker

The writable layer is ephemeral. If the container is removed, all data stored in that layer disappears as well.  
This is not suitable for persistent application data such as databases or uploaded files.  
To persist data independently of the container lifecycle, Docker provides volumes.

---

#### Named volume vs bind mount

The command `docker run -v volume_name:/var/lib/mysql mysql` mounts the volume named `volume_name` on the container. If the volume does not already exist, Docker creates it automatically before mounting it.  
The data used in named volumes, like the one above, is stored inside the folder `/var/lib/docker/volumes`.  
This is called **volume mount**.

In case we want to instead store the container's data in a specific location on the Docker host (a bind mount) rather than in a named volume, we can provide the host directory path directly, like `docker run -v /data/mysql:/var/lib/mysql mysql`.  
In case a file or directory provided does not yet exist on the Docker host, Docker automatically creates the directory on the host for you.  
This is called **bind mount**.  
Unlike named volumes, bind mounts rely on a specific path on the Docker host.

> [!IMPORTANT]
>
> The writable container layer is intended for temporary runtime data only.  
> Any important application data should be stored in a Docker volume or bind mount; otherwise it will be lost when the container is removed.

Another important thing about mounting is that the syntax `--mount` is generally preferred over `-v` because it is more explicit and supports all the available options.  
With `--mount`, the commands above can be rewritten as:

- `docker run --mount type=volume,src=volume_name,dst=/var/lib/mysql mysql`;
- `docker run --mount type=bind,source=/data/mysql,target=/var/lib/mysql mysql`.

---

#### Storage Drivers vs Volume Drivers

At this point, it is important to distinguish between two different concepts:

- **Storage Drivers**: manage image layers and the writable container layer;
- **Volume Drivers**: manage persistent volumes independently of the container filesystem.

The copy-on-write (CoW) mechanism used by Docker images and containers is managed by the **Storage Drivers**.  
Docker automatically selects the most appropriate storage driver supported by the host operating system.

Volumes, instead, are not handled by storage drivers but by **Volume driver plugins**.  
Volume drivers are responsible only for managing volumes; they are independent from the storage driver used for image layers.  
A Volume Driver plugin can also provision storage on external systems instead of using the local filesystem.  
The default volume driver plugin is `local` but there are many others that we can choose, including the cloud provider ones.

---

### Volumes in Kubernetes

The Pods created in Kubernetes are transient in nature.  
To persist data processed by the Pod, we attach a volume to them.  
Even if the pod is deleted, the data generated or processed remains on the volume.

In the following example, we create a pod and provide it a volume to store the processed data.  
We would normally back this volume with proper persistent storage, but to keep things simple for now, we'll use `hostPath` (which is not suitable for critical environments).  
The possible values for `spec.volumes[*].hostPath.type` are

- `Directory`
- `DirectoryOrCreate`
- `File`
- `FileOrCreate`
- `Socket`
- `CharDevice`
- `BlockDevice`

The `spec.containers[*].volumeMounts[*].mountPath` declares where the volume is mounted within the container.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: random-number-generator
spec:
  containers:
  - name: alpine
    image: alpine
    command: ["/bin/sh", "-c"]
    args: ["shuf -i 0-100 -n 1 >> /opt/number.out;"]
    volumeMounts:
    - name: data-volume
      mountPath: /opt
  volumes:
  - name: data-volume
    hostPath:
      path: /data
      type: Directory
```

From now on, any file created in the volume will be stored in the directory `/data` of the node.

> [!NOTE]
>
> `volumeMounts` is for the container while `volumes` is for the pods.
>
> ```text
> Pod
> │
> ├── volumes
> │     └── data-volume
> │
> └── containers
>       └── volumeMounts
>             name: data-volume
> ```

The `hostPath` volume works fine if we are on a single-node cluster; however, it is not recommended for use in a multi-node cluster because not all the nodes have the same contents of the path specified in `hostPath.path` (unless we set up a specific solution like a replicated cluster storage solution).

> [!WARNING]
>
> `hostPath` is mainly intended for testing, development or special system workloads.  
> It tightly couples a Pod to a specific node because the data exists only on that node.  
> Production applications should normally use PersistentVolumes backed by network or cloud storage instead.

Kubernetes supports many storage backends, traditionally including NFS, Fibre Channel and cloud-provider block storage.  
Modern Kubernetes clusters generally access these backends through CSI drivers.

---

### Persistent Volumes

In the previous examples we defined volumes inside Pod manifests.  
This is not ideal in scenarios when we have to deploy multiple pods and have many deployments to manage.  
We want to avoid a situation where:

- the user has to define and configure volumes in each Pod definition file;
- we have to redefine multiple Pod definition files each time a modification needs to be made to the volume.

Instead, we want to manage the volumes centrally in a way that an administrator can create a large pool of storage and the users carve out pieces from it as required.

PersistentVolumes help us accomplish all of this.  
A `PersistentVolume` represents a piece of storage in the cluster, normally provisioned by an administrator; the collection of all PersistentVolumes forms the pool of storage available to the cluster.  
The users can now select storage from this pool using `PersistentVolumeClaims`.

Here is an example manifest, where we define the `accessModes`, the `capacity` and use `hostPath` as the volume type

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv-vol1
spec:
  accessModes:
  - ReadWriteOnce
  capacity:
    storage: 1Gi
  hostPath:
    path: /tmp/data
```

The Access Mode defines how a volume should be mounted on a host and can be:

- `ReadOnlyMany`: the volume can be mounted as read-only by many nodes;
- `ReadWriteOnce`: the volume can be mounted as read-write by a single node;
- `ReadWriteOncePod`: the volume can be mounted as read-write by a single pod;
- `ReadWriteMany`: the volume can be mounted as read-write by many nodes.

> [!CAUTION]
>
> A PersistentVolume is not the storage itself, but a Kubernetes abstraction representing a piece of storage made available to the cluster.

```text
         Pod
          │
    volumeMounts
          │
          ▼
+------------------+
|       PVC        |
|  (user request)  |
+------------------+
          │ binds to
          ▼
+------------------+
|       PV         |
| (cluster storage)|
+------------------+
          │
          ▼
NFS / EBS / Azure Disk /
Ceph / NetApp / CSI ...
```

---

### Persistent Volume Claims

A `PersistentVolumeClaim` is a request for storage made by a user.  
Pods use PVCs to access `PersistentVolumes`.

> [!NOTE]
>
> Pods never reference a PersistentVolume directly.  
> Instead, they reference a PersistentVolumeClaim, which abstracts the underlying storage implementation.

At any given time, a `PersistentVolumeClaim` is bound to exactly one `PersistentVolume`, and a bound `PersistentVolume` can be claimed by only one `PersistentVolumeClaim`.  
During the binding process, Kubernetes looks for a PersistentVolume that satisfies all the requirements requested by the claim, including storage capacity, access modes, volume mode, StorageClass and selector (if specified).  
If there are multiple possible matches for a single claim and you want to target a particular volume, you can use Labels and Selectors to bind to the right one.  

```yaml
spec:
  selector:
    matchLabels:
      type: fast
```

Finally note that a smaller claim may get bound to a larger volume if all the other criteria match and there are no better options; in this case, no other claim can use the remaining capacity in the volume.  
If there are instead no volumes available, the PVC remains in a `Pending` state until newer volumes are made available to the cluster or until a StorageClass dynamically provisions one.

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: myclaim
spec:
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 500Mi
```

```console
$ kubectl create -f pvc-definition.yaml
persistentvolumeclaim/myclaim created

$ kubectl get pvc myclaim
NAME      STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   VOLUMEATTRIBUTESCLASS   AGE
myclaim   Bound    pvc-c192c2c7-299e-44bf-b60c-b8dccaaa772c   500Mi      RWO            standard       <unset>                 16s
```

When a `PersistentVolumeClaim` is deleted, the action taken on the underlying `PersistentVolume` depends on the `PersistentVolume`'s `persistentVolumeReclaimPolicy`:

- `Retain` (default for manually created PersistentVolumes): the `PersistentVolume` is released but not deleted. Manual cleanup is required before it can be reused;
- `Delete`: the PersistentVolume and the underlying storage are automatically deleted (if supported by the storage provider);
- `Recycle`: the volume contents are deleted and the `PersistentVolume` becomes available again. This policy has been deprecated.

> [!NOTE]
>
> Deleting a PersistentVolumeClaim does not necessarily guarantee that the underlying data is securely erased.  
> Depending on the storage backend, proper cleanup may require operations such as unmounting, detaching, filesystem reformatting, snapshot removal, encryption key rotation or other provider-specific actions.

Once the PVC has been created, we can use it in a Pod definition file by specifying its name under the `persistentVolumeClaim` section of the `volumes` field, like this:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: mypod
spec:
  containers:
    - name: myfrontend
      image: nginx
      volumeMounts:
      - mountPath: "/var/www/html"
        name: mypd
  volumes:
    - name: mypd
      persistentVolumeClaim:
        claimName: myclaim
```

The same is true for ReplicaSets or Deployments. Add this to the pod template section of a `Deployment` or `ReplicaSet`.

---

### Storage Classes

Before StorageClasses were introduced, persistent storage was usually statically provisioned.  
Every time an application required storage, a cluster administrator had to:

- provision a storage volume on the underlying infrastructure (for example an AWS EBS volume or a GCE Persistent Disk);
- create a matching PersistentVolume object pointing to that storage.

This approach is called static provisioning.

With Storage Classes instead, you can perform dynamic provisioning, which means provisioning volumes automatically when the application requires it.

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: ebs-rwo
provisioner: ebs.csi.aws.com
reclaimPolicy: Retain
volumeBindingMode: WaitForFirstConsumer
parameters:
  type: gp3
```

> [!NOTE]
>
> Modern Kubernetes clusters usually provision storage through CSI (Container Storage Interface) drivers.  
> Older in-tree provisioners such as `kubernetes.io/gce-pd` or `kubernetes.io/aws-ebs` are deprecated and remain only for backward compatibility.
>
> Modern CSI drivers may also support VolumeSnapshots, allowing point-in-time copies of Persistent Volumes for backup and restore operations.

With this approach, you don't need to manually create a `PersistentVolume` definition, because the CSI provisioner creates the PV automatically.

In the example we used the EBS provisioner on AWS but there are many others, like GCE PD, Azure File, Azure Disk, CephFS, Portworx, ScaleIO.  
The `parameters` field is very specific to the provisioner used.  
There is also the `volumeBindingMode` field, which can be:

- `Immediate`: (default for many older StorageClasses) provisions the volume as soon as the PVC is created;
- `WaitForFirstConsumer`: delays provisioning until a Pod actually uses the PVC, allowing Kubernetes to select a storage volume in the same availability zone as the scheduled Pod.

For the `PersistentVolumeClaim` to use the `StorageClass`, we need to specify the StorageClass name in the PVC definition file.

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: myclaim
spec:
  accessModes:
  - ReadWriteOnce
  storageClassName: ebs-rwo
  resources:
    requests:
      storage: 500Mi
```

> [!IMPORTANT]
>
> A `StorageClass` does not create storage by itself.  
> Storage is dynamically provisioned only when a `PersistentVolumeClaim` referencing that `StorageClass` is created
>
> When the PVC is created, Kubernetes asks the provisioner specified in the `StorageClass` to allocate a new storage volume.  
> The provisioner creates the underlying storage resource (for example an AWS EBS volume), automatically creates the corresponding `PersistentVolume`, and binds it to the PVC.

| Static Provisioning     | Dynamic Provisioning       |
| ----------------------- | -------------------------- |
| Admin creates disk      | Disk created automatically |
| Admin creates PV        | PV created automatically   |
| PVC binds existing PV   | PVC triggers provisioning  |
| Manual management       | Automatic management       |
| More operational effort | Recommended approach       |

---

### StatefulSets

A StatefulSet runs a group of Pods, and maintains a sticky identity for each of those Pods. This is useful for managing applications that need persistent storage or a stable, unique network identity.

With StatefulSets we have:

- pods created in sequential order: after the first pod is deployed, it has to be in running and ready state before the next pod is deployed;
- stable hostname and stable DNS name: each pod is assigned a unique, ordinal and predictable name based on the ordinal index of each pod (starting with 0 for the first pod and incrementing by one for each pod);
- stable storage: each Pod in a `StatefulSet` can automatically receive its own `PersistentVolumeClaim` through `volumeClaimTemplates`.

StatefulSets are useful when:

- you need pods with a stable, predictable hostname and pod name;
- you need to have a specific startup sequence for pods inside the set.

This is useful for stateful applications such as databases or distributed systems where each instance may need a stable identity and persistent storage.  
For example, a replicated database may designate one Pod as primary and others as replicas. The database software is responsible for configuring replication, while the `StatefulSet` provides the stable identities, network names and persistent storage that make this architecture possible.

> [!IMPORTANT]
>
> A StatefulSet provides stable identity and storage, but it does not provide database replication, backup, consistency or disaster recovery.  
> These capabilities must be provided by the application/database or by additional Kubernetes/storage components.

```text
mysql-0  -> primary
mysql-1  -> replica
mysql-2  -> replica
```

This does not mean that Kubernetes itself makes mysql-0 the primary. The database software must implement this logic.

A `Deployment` does not guarantee stable Pod identities, ordered deployment, or dedicated persistent storage for each primary or replica; moreover, if the primary pod crashes and gets recreated, it will be assigned a new Pod name, hostname and IP.

The manifest of a `StatefulSet` is similar to the one of a `Deployment`.

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: mysql
  labels:
    app: mysql
spec:
  template:
    metadata:
      labels:
        app: mysql
    spec:
      containers:
      - name: mysql
        image: mysql
  replicas: 3
  selector:
    matchLabels:
      app: mysql
  serviceName: mysql-h
  volumeClaimTemplates:
  - metadata:
      name: mysql-data
    spec:
      accessModes:
      - ReadWriteOnce
      resources:
        requests:
          storage: 10Gi
```

A StatefulSet does not require `volumeClaimTemplates`. They are used when each Pod needs its own persistent storage (see later).

When scaling up a `StatefulSet`, each newly created pod has to be in running and ready state before the next pod is deployed.  
This is useful for example when scaling a MySQL DBMS, where each new instance can clone from the previous one.  
It works in reverse order when scaling down: the last instance is removed first, followed by the second-to-last one.  
The same is true on termination: when deleting a `StatefulSet`, the pods are deleted in reverse order.  
In any case, this is the default behavior of StatefulSets, which we can override to not follow an ordered launch but still have the other benefits of a StatefulSet (like a stable and unique network ID). For that we need to set `spec.podManagementPolicy` to `Parallel` (the default is `OrderedReady`).

---

#### Headless Service in StatefulSets

StatefulSets typically use a Headless `Service` to provide stable network identities and DNS records for individual Pods.  
This is because a Headless `Service`:

- does not load-balance traffic;
- does not allocate a ClusterIP;
- provides a stable DNS record for every `Pod` instead of a single virtual IP, allowing each `Pod` to be reached individually.

Instead of providing a single ClusterIP, DNS for a Headless `Service` resolves to the IP addresses of its Pods, formed as `<pod_name>.<headless_service_name>.<namespace>.<cluster_domain>`.  
This is particularly useful for clustered databases such as MySQL, PostgreSQL, Cassandra, ZooKeeper or Kafka, where each node must be individually addressable.

```text

           Headless Service
                mysql-h
                    │
      ┌─────────────┼─────────────┐
      │             │             │
mysql-0         mysql-1       mysql-2
      │             │             │
10.0.1.15     10.0.1.22     10.0.1.30

mysql-0.mysql-h.default.svc.cluster.local ──► 10.0.1.15

mysql-1.mysql-h.default.svc.cluster.local ──► 10.0.1.22

mysql-2.mysql-h.default.svc.cluster.local ──► 10.0.1.30
```

A Headless Service is created by setting `clusterIP: None`.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: mysql-h
spec:
  clusterIP: None
  selector:
    app: mysql
  ports:
    - port: 3306
```

For ordinary Pods, Kubernetes creates per-Pod DNS records only if the optional `hostname` and `subdomain` fields are specified.

```yaml
spec:
  subdomain: mysql-h
  hostname: mysql-pod
```

With StatefulSets, this configuration is performed automatically:

- the Pod hostname is the Pod name (`mysql-0`, `mysql-1`, ...);
- the subdomain is taken from `.spec.serviceName`, which must reference the Headless Service.

---

#### Storage in StatefulSets

The `volumeClaimTemplates` field provides stable storage using PersistentVolumes provisioned automatically (for example via a `StorageClass` provisioner).  
Unlike `Deployments`, where Pods may share or recreate storage arbitrarily, every `StatefulSet` `Pod` keeps its own dedicated volume even if it is rescheduled to another node.

```text
    Headless Service
            │
   ┌────────┼───────┐
   │        │       │
mysql-0  mysql-1  mysql-2
   │        │       │
 PVC-0    PVC-1   PVC-2
   │        │       │
 PV-0      PV-1    PV-2
```

This mechanism allows StatefulSets to automatically create one `PersistentVolumeClaim` for each Pod.  
Instead of manually creating PVCs, we define a `volumeClaimTemplates` section; then, every Pod receives its own dedicated `PersistentVolumeClaim`.

Example:

```yaml
volumeClaimTemplates:
- metadata:
    name: data
  spec:
    accessModes:
    - ReadWriteOnce
    resources:
      requests:
        storage: 5Gi
```

If the StatefulSet has three replicas, Kubernetes automatically creates:

```text
data-mysql-0

data-mysql-1

data-mysql-2
```

Each Pod keeps its own PVC, and Kubernetes reattaches the corresponding volume when the Pod is recreated or rescheduled, provided that the storage backend supports the required topology and access mode.

---

#### Deployment vs StatefulSet

Unlike in `Deployments`, Pods belonging to a `StatefulSet` are not interchangeable. Each Pod keeps its own identity, storage and network name across restarts.

| Feature               | Deployment         | StatefulSet          |
| --------------------- | ------------------ | -------------------- |
| Stable Pod names      | :x:                | :white_check_mark:   |
| Stable DNS names      | :x:                | :white_check_mark:   |
| Dedicated PVC per Pod | :x:                | :white_check_mark:   |
| Ordered startup       | :x:                | :white_check_mark: ¹ |
| Ordered termination   | :x:                | :white_check_mark: ¹ |
| Parallel replicas     | :white_check_mark: | Optional             |
| Stateless apps        | :white_check_mark: | :warning:            |
| Stateful apps         | :warning: ²        | :white_check_mark:   |

¹ Default behavior; can be changed with `podManagementPolicy: Parallel`.  
² Databases and other stateful workloads → commonly `StatefulSet`.

---

## Security

---

### Kubernetes Security Primitives

Since Kubernetes is a production-ready platform for hosting applications, security is one of the primary concerns.

It is vital to **secure the underlying hosts**, that is, the physical or virtual hosts that form the cluster. Ensure all host access is protected by:

- disabling root access;
- disabling password-based authentication;
- enforcing SSH key-based authentication;
- implementing other critical security measures.

The first line of defense is controlling access to the kube-apiserver itself.  
We need to make two types of decisions:

- who can access the cluster: defined through the **authentication mechanism** like certificates, tokens, ServiceAccounts (for machines), external authentication providers (like LDAP);
- what can they do: defined through the **authorization mechanism** like RBAC, ABAC (where A stands for attributes), Node Authorization, Webhook Mode.

Then, all communication between the kube-apiserver and the other components of the cluster is secured by TLS encryption.

For communication within the cluster, we have already seen that, by default, all pods can access all other pods, but we can restrict access between them using **NetworkPolicies**.

---

### Authentication

Our concern is to manage authentication for users (cluster administrators and developers) and Service Accounts.

Kubernetes does not manage user accounts natively.  
So we cannot create or view the list of users with `kubectl`.  
Instead, it relies on external sources like:

- static password and token files (with tokens and user details);
- certificates;
- third-party authentication protocols (identity providers like LDAP, Kerberos, etc.);

This is different for ServiceAccounts: in this case you can manage them via the API.

Focusing on users, all user access is managed by the `kube-apiserver`, whether accessing the cluster through:

- the `kubectl`;
- the API directly (like `curl https://kube-server-ip:6443/`).

In the approach with static password and token files, we have CSV files like the following one `user-token-details.csv`:

```csv
token,user,uid,groups
"v7f89as7df89as7df89as7","mario.rossi","1001","developers,engineering"
"9a8b7c6d5e4f3g2h1i0jkl","giulia.bianchi","1002","administrators,security"
"z1x2c3v4b5n6m7q8w9e0rt","luca.verdi","1003","viewers,analytics"
"p0o9i8u7y6t5r4e3w2q1as","app-service-account","2001","system-agents"
```

with this, you can pass the token as a Bearer token in the request's Authorization header:

```bash
curl -v -k https://master-node-ip:6443/api/v1/pods --header "Authorization: Bearer v7f89as7df89as7df89as7"
```

This method is not a recommended authentication mechanism because it stores tokens or passwords in clear text.

---

### KubeConfig

When communicating with a Kubernetes cluster, authentication and connection parameters may need to be provided to both `kubectl` and `curl`.  
For example:

```bash
curl https://my-kube-playground:6443/api/v1/pods \
  --key admin.key \
  --cert admin.crt \
  --cacert ca.crt
```

```bash
kubectl get pods \
  --server https://my-kube-playground:6443 \
  --client-key admin.key \
  --client-certificate admin.crt \
  --certificate-authority ca.crt
```

In order to avoid typing all those each time, we can move this information to a configuration file called `kubeconfig` and specify only this one with the `kubeconfig` option in the command.

```bash
kubectl get pods \
  --kubeconfig config
```

By default, the `kubectl` tool looks for the file `~/.kube/config`.  
So, if the file is placed there, we don't have to specify the path to that file each time.  
This default can be overridden by setting the `KUBECONFIG` environment variable, which specifies the kubeconfig file(s) to use.

```bash
kubectl get pods
```

That's the reason we didn't have to specify any options for our `kubectl` commands so far.

A kubeconfig file has a specific structure and mainly contains three collections:

- **Clusters**: the Kubernetes clusters that we need access to (for example, different environments, organizations or cloud providers);
- **Users**: the identities or credentials used to authenticate to those clusters;
- **Contexts**: define which user accesses which cluster and, optionally, which namespace is used by default.

Using a kubeconfig compared to using the parameters in the commands performed above:

- the server specification (`--server` or URL on curl) and CA certificate (`--certificate-authority` or `--cacert` on curl) go in the cluster section;
- the user's keys (`--client-key` or `--key` on curl) and certificates (`--client-certificate` or `--cert` on curl) go into the user section;
- combining them together, as you would when running a command, corresponds to the context.

```text
Cluster
  └── server + CA

User
  └── client certificate + client key

Context
  └── Cluster + User + Namespace
```

Here is an example of real kubeconfig file

```yaml
apiVersion: v1
kind: Config

clusters:
- name: development
  cluster:
    server: https://development.example.com:6443
    certificate-authority: /etc/kubernetes/pki/development/ca.crt
- name: my-kube-playground
  cluster:
    server: https://my-kube-playground.example.com:6443
    certificate-authority-data: LS0tLS1CRUdJTiBDRVJUSUZJQ0FURS0tLS0tCk1JSUIwekNDQVhTZ0F3SUJBZ0lSQU5DNE5qczJiWG8w
- name: production
  cluster:
    server: https://production.example.com:6443
    certificate-authority: /etc/kubernetes/pki/production/ca.crt
- name: google
  cluster:
    server: https://google.example.com:6443
    certificate-authority-data: LS0tLS1CRUdJTiBDRVJUSUZJQ0FURS0tLS0tCk1JSURYVENDQWtHZ0F3SUJBZ0lKQUwxYzJ2O

users:
- name: admin
  user:
    client-certificate: admin.crt
    client-key: admin.key
- name: my-kube-admin
  user:
    token: <token>
- name: dev-user
  user:
    token: <token>

contexts:
- name: admin@production
  context:
    cluster: production
    user: admin
- name: my-kube-admin@my-kube-playground
  context:
    cluster: my-kube-playground
    user: my-kube-admin
- name: dev@google
  context:
    cluster: google
    user: dev-user
    namespace: invoicing

current-context: admin@production
```

> [!NOTE]
>
> The CA of the destination server can be specified:
>
> - as a path to a certificate file, with the `certificate-authority` field;  
> - directly in the kubeconfig as a base64-encoded value, with the `certificate-authority-data` field.

![kubeconfig: clusters, context and users](./images/01-kubeconfig-clusters-context-users.png "kubeconfig contents")

```mermaid
flowchart LR

subgraph C["Clusters"]
    C1["Development"]
    C2["MyKubePlayground"]
    C3["Production"]
    C4["Google"]
end

subgraph CT["Contexts"]
    CT1["Admin@Production"]
    CT2["MyKubeAdmin@MyKubePlayground"]
    CT3["Dev@Google"]
end

subgraph U["Users"]
    U1["Admin"]
    U2["MyKubeAdmin"]
    U3["Dev User"]
    U4["Prod User"]
end

CT1 -->|cluster| C3
CT1 -->|user| U1

CT2 -->|cluster| C2
CT2 -->|user| U2

CT3 -->|cluster| C4
CT3 -->|user| U3

K["current-context"] --> CT1
```

```text
Clusters                  Contexts                           Users

Development

MyKubePlayground ◄──────── MyKubeAdmin@MyKubePlayground ───► MyKubeAdmin

Production ◄────────────── Admin@Production ───────────────► Admin

Google ◄────────────────── Dev@Google ─────────────────────► Dev User

                                                             Prod User
```

As you can notice, there is also a `current-context` field inside the kubeconfig.  
It tells `kubectl` which context to use by default when no other context is explicitly specified.

The `current-context` field is optional.

Useful commands:

- `kubectl config view`: view the configuration of the currently used kubeconfig file;
- `kubectl config view --kubeconfig=mycustom-config`: view the configuration of a specific kubeconfig file;
- `kubectl config get-clusters`: list the clusters available;
- `kubectl config get-users`: list the users available;
- `kubectl config get-contexts`: list the contexts available;
- `kubectl config current-context`: view the current context being used;
- `kubectl config use-context <context-name>`: change the current context of the current kubeconfig file.

```console
$ kubectl config current-context
admin@production

$ kubectl config use-context dev@google
Switched to context "dev@google".

$ kubectl config current-context
dev@google
```

---

### API Groups

Whatever operations we have performed so far with the cluster, we have been interacting with the Kubernetes API Server, either through the `kubectl` utility or directly through its REST API. For example:

- for checking the version we can use `curl https://kube-master:6443/version`;
- for many operations, we use the path `/api`, for example `curl https://kube-master:6443/api/v1/pods`;
- for many other operations, we need to use the path `/apis`, for example `curl https://kube-master:6443/apis/apps/v1/deployments`.

The Kubernetes API is organized into API groups, which provide a way to logically group related resources and make the API easier to extend.  
API groups are accessible via API paths in the following way:

- `/api`: the API path for the core API group `/api/v1`, which contains resources such as `Pods`, `Services`, `Nodes`, `Namespaces`, `ConfigMaps`, `Secrets`, `PersistentVolumes` and `PersistentVolumeClaims`;
- `/apis`: the API path that contains named API groups, which organize related resources. It has groups like:
  - `/apps/v1`: `Deployments`, `ReplicaSets`, `StatefulSets`;
  - `/batch/v1`: `Jobs`, `CronJobs`;
  - `/networking.k8s.io/v1`: `NetworkPolicies`, `Ingress`;
  - `/storage.k8s.io/v1`: `StorageClasses`;
  - `/authentication.k8s.io/v1`: authentication-related APIs;
  - `/rbac.authorization.k8s.io`: `Roles`, `RoleBindings`, `ClusterRoles`, `ClusterRoleBindings`;
  - `/certificates.k8s.io/v1`: `CertificateSigningRequests`;
- Other API server endpoints (that are neither API groups nor part of one), representing `nonResourceURLs` like:
  - `/version`: to view the version of the cluster;
  - `/healthz`: to monitor the health of the cluster;
  - `/metrics`: to retrieve metrics on the cluster;
  - `/logs`: to provide access to log files exposed by the API server.

> [!CAUTION]
>
> Not every endpoint exposed by the API server represents a Kubernetes API group.  
> For example, `/version`, `/metrics` and `/healthz` are non-resource endpoints.

The resources are exposed through `/api` and `/apis` API paths.

| Component        | URI                                  |
| ---------------- | ------------------------------------ |
| Core API groups  | `/api/<version>/<resource>`          |
| Named API groups | `/apis/<group>/<version>/<resource>` |
| Other endpoints  | `/<endpoint>`                        |

> [!NOTE]
>
> Keep in mind that **resource** and **kind** are not the same thing. For example:
>
> ```text
> Resource: deployments
> Kind:     Deployment
> ```
>
> the command `kubectl api-resources` highlights this difference
>
> ```console
> $ kubectl api-resources
> NAME           SHORTNAMES   APIVERSION              NAMESPACED   KIND
> pods           po           v1                      true         Pod
> services       svc          v1                      true         Service
> deployments    deploy       apps/v1                 true         Deployment
> statefulsets   sts          apps/v1                 true         StatefulSet
> jobs                        batch/v1                true         Job
> cronjobs       cj           batch/v1                true         CronJob
> ingresses      ing          networking.k8s.io/v1    true         Ingress
> nodes          no           v1                      false        Node
> ```

| Kind                  | Resource               | API group         | API version | API path                                     |
| --------------------- | ---------------------- | ----------------- | ----------- | -------------------------------------------- |
| Pod                   | pods                   | core              | v1          | `/api/v1/pods`                               |
| Service               | services               | core              | v1          | `/api/v1/services`                           |
| Deployment            | deployments            | apps              | v1          | `/apis/apps/v1/deployments`                  |
| StatefulSet           | statefulsets           | apps              | v1          | `/apis/apps/v1/statefulsets`                 |
| NetworkPolicy         | networkpolicies        | networking.k8s.io | v1          | `/apis/networking.k8s.io/v1/networkpolicies` |
| PersistentVolumeClaim | persistentvolumeclaims | core              | v1          | `/api/v1/persistentvolumeclaims`             |
| StorageClass          | storageclasses         | storage.k8s.io    | v1          | `/apis/storage.k8s.io/v1/storageclasses`     |

```text
Kubernetes API Server
│
├── API discovery / resource APIs
│   ├── /api/v1
|   |        ├── pods
|   |        ├── services
|   |        ├── namespaces
|   |        ├── configmaps
|   |        ├── secrets
|   |        ├── nodes
|   |        └── ...
│   │
│   └── /apis
│       ├── /apps/v1
│       ├── /batch/v1
│       ├── /networking.k8s.io/v1
│       ├── /storage.k8s.io/v1
│       ├── /authentication.k8s.io/v1
│       ├── /certificates.k8s.io/v1
│       └── ...
│
└── Other API server endpoints (nonResourceURLs)
    ├── /version
    ├── /healthz
    ├── /metrics
    ├── /logs
    └── ...
```

You can discover the available API groups through the API Server itself, normally with the commands:

- `curl https://localhost:6443 -k`;
- `curl https://localhost:6443/api -k`;
- `curl https://localhost:6443/apis -k`;

and if you want to check a specific API, you can perform `curl https://localhost:6443/apis/apps/v1 -k`.

In order to avoid specifying authentication parameters, you can start a kubectl-proxy with the command

```bash
kubectl proxy
```

and then perform commands like:

- `curl http://localhost:8001`;
- `curl http://localhost:8001/api`;
- `curl http://localhost:8001/apis`.

and you can finally get specific information on a single API with a command like `curl http://localhost:8001/apis/apps/v1`.

> [!TIP]
>
> `kubectl proxy` starts a local proxy that forwards requests to the Kubernetes API server.  
> By default, it listens on `127.0.0.1:8001` and uses the current kubeconfig context to determine how to connect to the API server.
> Using this proxy, we don't have to specify credentials and certificates in each curl command.

The API group and version are also reflected in the `apiVersion` field of Kubernetes objects:

- Core API group, uses `apiVersion: v1`

  ```yaml
  apiVersion: v1
  kind: Pod
  ```

- Named API group, uses `apiVersion: <group>/<version>`

  ```yaml
  apiVersion: apps/v1
  kind: Deployment
  ```

> [!CAUTION]
>
> The API group and API version are separate concepts.  
> For example, `apps/v1` refers to the `apps` API group and its `v1` version.  
> For resources in the core API group, the group name is omitted and only the version is specified, such as `v1`.

Each resource supports a set of operations, commonly referred to as Kubernetes verbs, such as `get`, `list`, `create`, `update`, `patch`, `delete` and `watch`.

> [!NOTE]
>
> Kubernetes verbs should not be confused with HTTP methods.  
> For example
>
> - `kubectl get pods` uses the Kubernetes `get` verb, which is implemented through an HTTP `GET` request;
> - `kubectl create deployment ...` uses the Kubernetes `create` verb and is implemented using an HTTP `POST` request;
> - `watch` is implemented using an HTTP `GET` request with the `?watch=true` parameter.

To summarize:

- the Kubernetes API Server exposes Kubernetes resources through API groups;
- at the top level we have a core API group (exposed through `/api`) and named API groups (exposed through `/apis`);
- under the named API groups we have one API group for each API section;
- each API group can have one or more API versions;
- each API version exposes one or more resources;
- resources support operations (Kubernetes verbs) such as `get`, `list`, `create`, `update`, `patch`, `delete` and `watch`.

```text
                Kubernetes API Server
                        │
        ┌───────────────┴───────────────┐
        │                               │
    Core API group                Named API groups
      /api                           /apis
        │                               │
        v1                     ┌────────┼────────────────┐
        │                      │        │                │
┌───────┼────────┐          apps/v1   batch/v1   networking.k8s.io/v1
│       │        │             │        │                │
pods  services  configmaps    v1       v1               v1
                               │
                         deployments
                               │
                         Kubernetes verbs
                   ┌───────────┼───────────┐
                 get        create       delete
```

```text
API group
    │
    └── API version
            │
            └── Resource
                    │
                    └── Operations / verbs
```

Useful commands:

- `kubectl api-versions`: show available API group and version;
- `kubectl api-resources`: show available resources and the corresponding API group (especially useful when you can't find a resource on the documentation pages);
- `kubectl explain <resource name>`: gives details on the Kind and lists the top level fields and their type;
- `kubectl explain <resource name>.<field name>`: gives details on the Kind and lists the subfields and their type;
- `kubectl explain <resource name> --recursive`: lists all fields that we would put in the YAML file.

---

### Authorization

Authentication answers **"Who are you?"**
Authorization answers **"What are you allowed to do?"**

We need authorization in a cluster because we want to differentiate the actions that different users can perform. For example:

- we don't want non-administrative users to perform operations like adding or deleting nodes, or modifying network or storage configuration;
- we want to provide `ServiceAccounts` with the minimum level of access required to perform their operations;
- we want to restrict access for users or groups to specific namespaces, in order to protect the workloads running in each of them.

Authorization defines **what users are allowed to do**.

There are different authorization modes / authorizers:

| Authorizer      | Main purpose                                                            | Configuration                          |
| --------------- | ----------------------------------------------------------------------- | -------------------------------------- |
| **Node**        | Authorize kubelets based on the Pods and Nodes they are associated with | API server authorization configuration |
| **ABAC**        | Authorize users/groups based on policy attributes                       | Policy file                            |
| **RBAC**        | Authorize users/groups/ServiceAccounts through Roles and Bindings       | Kubernetes API resources               |
| **Webhook**     | Delegate authorization decisions to an external service                 | External authorization service         |
| **AlwaysAllow** | Allow all requests                                                      | kube-apiserver                         |
| **AlwaysDeny**  | Deny all requests                                                       | kube-apiserver                         |

The modes are set using the `--authorization-mode` option on the kube-apiserver.  
If this option is not specified, it is set to `AlwaysAllow` by default.

> [!NOTE]
>
> In practice, production clusters normally configure a more restrictive authorization mode, commonly `Node,RBAC`.  
> The `AlwaysAllow` default applies when no authorization mode is explicitly configured.

You can check the authorization mode configured with the following command:

```bash
# From kubectl
kubectl describe pod kube-apiserver-controlplane -n kube-system | grep authorization-mode

# Alternatively, if on the control-plane with the kube-apiserver
cat /etc/kubernetes/manifests/kube-apiserver.yaml | grep authorization-mode
```

> [!TIP]
>
> Replace `kube-apiserver-controlplane` with the name of the API Server `Pod` in your instance (for example `kube-apiserver-minikube` in Minikube)

You may also provide a comma separated list of multiple modes to use, like `--authorization-mode=Node,RBAC,Webhook`.  
When having multiple modes configured, the request is authorized using each authorizer in the specified order.  
Each authorizer can return:

- `Allow`: the request is immediately authorized;
- `Deny`: the request is immediately denied;
- `No opinion`: the request is forwarded to the next authorizer.

If all authorizers return `No opinion`, the request is denied.

In the example above, if a user sends a request, it is first handled by the Node authorizer which returns `No opinion` because it only handles node requests; then the request is forwarded to the RBAC module, which performs its checks and returns `Allow`. The authorization is now complete and the user is given access to the requested resource.

> [!TIP]
>
> In modern Kubernetes environments, RBAC is by far the most common authorization mechanism for users, groups and ServiceAccounts.
>
> The Node authorizer handles kubelet authorization, while Webhook authorization can be used when an organization needs an external policy engine or centralized authorization system.

The **Node authorizer** specifically authorizes API requests **made by kubelets**.

We know that the `kube-apiserver` is accessed by users as well as by the kubelets running on the nodes within the cluster.  
The kubelet:

- accesses the API server to read information about Services, Endpoints, Nodes and other information about Pods assigned to it;
- reports information to the API server about the node, such as its status, the status of the Pods running on it and events.

The Node authorizer authorizes kubelets to perform the operations required to manage their assigned Pods and nodes.

```text
- read
  - Services
  - Endpoints
  - Nodes
  - Pods
  - Secrets
  - ConfigMaps
  - PVCs
  - PVs

- write
  - Node status
  - Pod status
  - Events

and some other resources
```

> [!TIP]
>
> The Node authorizer works together with the NodeRestriction admission plugin to further restrict what a kubelet can modify.

To be authorized by the Node authorizer, a kubelet must authenticate with an identity using the `system:node:<node-name>` username format and belong to the `system:nodes` group.  
The Node authorizer then grants the kubelet the permissions required to operate its assigned Pods and Node.

**Attribute-Based Access Control** is an authorization mechanism where we associate a user or a group of users with a set of permissions based on attributes.

> [!NOTE]
>
> ABAC permissions are expressed using the same Kubernetes API concepts introduced in the previous section: API groups, resources and verbs.

With ABAC, we create a policy file containing a set of policies defined in JSON format and pass it to the API server.
For example:

- This policy allows `dev-user` to perform read-only operations on `Pods` in any namespace.

  ```json
  {"apiVersion":"abac.authorization.kubernetes.io/v1beta1","kind":"Policy","spec":{"user":"dev-user","namespace":"*","resource":"pods","readonly":true,"apiGroup":""}}
  ```

- This policy allows members of the `dev-users` group to access `Pods` in any namespace.

  ```json
  {"apiVersion":"abac.authorization.kubernetes.io/v1beta1","kind":"Policy","spec":{"group":"dev-users","namespace":"*","resource":"pods","apiGroup":"*"}}
  ```

- This policy allows members of the `security-1` group to access `CertificateSigningRequest` resources in any namespace.

  ```json
  {"apiVersion":"abac.authorization.kubernetes.io/v1beta1","kind":"Policy","spec":{"group":"security-1","namespace":"","resource":"certificatesigningrequests","apiGroup":"certificates.k8s.io"}}
  ```

> [!NOTE]
>
> The field `readonly: true` limits the policy to read operations such as `get`, `list` and `watch`.  
> Without `readonly`, a matching policy can also authorize write operations.

With ABAC, policies are configured through a local file on the API server, and changes require the API server to be restarted to take effect.  
This makes ABAC difficult to manage, especially in large environments.

> [!NOTE]
>
> ABAC is a legacy authorization mechanism and is generally not used for new Kubernetes deployments.  
> RBAC is the standard mechanism for managing permissions within Kubernetes.

**Role-Based Access Control** makes the management of permissions much easier.  
Instead of directly associating every user or group with a set of permissions, we define a **role** containing a set of rules and then associate users or groups with that role.

For example, we can create a role with a set of permissions for developers and then associate all developers with that role.  
Similarly, we can create a role for security users with the permissions required by their job and then associate all security users with that role.  
Whenever a change needs to be made to the permissions, we modify the role and the change applies to all users associated with it.  
RBAC provides a standard and flexible approach to managing access within Kubernetes.

RBAC is implemented in Kubernetes using four resources:

- `Role`
- `ClusterRole`
- `RoleBinding`
- `ClusterRoleBinding`

RBAC represents permissions as Kubernetes API objects. A `Role` or `ClusterRole` defines a set of permissions, while a `RoleBinding` or `ClusterRoleBinding` associates those permissions with users, groups or `ServiceAccounts`.

> [!NOTE]
>
> RBAC permissions are additive: there are no explicit deny rules in Kubernetes RBAC.

We'll see RBAC in more detail in the next chapters.

If we want to **outsource the authorization decision** to an external system instead of using only the built-in authorization mechanisms, we can use the `Webhook` authorizer.

For example, we can use a third-party policy engine such as **Open Policy Agent (OPA)**.

In this scenario, Kubernetes sends an authorization request to the external service containing information about the request, such as:

- the user or identity;
- the requested resource;
- the requested operation;
- the namespace;
- other relevant request attributes.

Then, the external authorization service evaluates the request according to its policies and returns a decision.

Based on that response, the Kubernetes API server either allows or denies the original request.

```mermaid
flowchart LR

U["User / Client<br/>(kubectl, application, etc.)"]
K["Kubernetes<br/>API Server"]
O["Open Policy Agent<br/>(OPA)"]

U -->|"1. API request"| K
K -->|"2. Authorization request<br/>(user, verb, resource, namespace, etc.)"| O
O -->|"3. Allow / Deny"| K
K -->|"4. Response<br/>(allow / deny)"| U
```

The flow can therefore be summarized as:

1. The user sends a request to the Kubernetes API server.
2. The API server forwards the authorization request to the external authorization service.
3. The external service evaluates the request according to its policies and returns an authorization decision.
4. The API server allows or denies the original request based on that decision.

> [!NOTE]
>
> A Webhook authorizer does not mean that Kubernetes delegates all authentication to the external service.
>
> The Webhook is specifically used as an **authorization mechanism**: Kubernetes asks the external service whether a particular request should be allowed or denied.

In addition to Authentication and Authorization, a request must also be admitted by the admission controller before it can be persisted:

- Authentication → Who are you?
- Authorization → What are you allowed to do?
- Admission → Is this request/object acceptable?

but Admission and Admission Controllers will be covered in more detail in the next section.

---

### Role-Based Access Control (RBAC)

We manage RBAC with the following objects:

- `Role`: defines permissions within a namespace;
- `ClusterRole`: defines permissions that can be applied at the cluster level via a `ClusterRoleBinding` or to a specific namespace via a `RoleBinding`;
- `RoleBinding`: assigns a `Role` or a `ClusterRole` to the namespace of the binding;
- `ClusterRoleBinding`: assigns a `ClusterRole` at the cluster level.

So, making an example, we first create a `Role` object

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: developer
  namespace: default
rules:
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["list", "get", "create", "update", "delete"]
- apiGroups: [""]
  resources: ["configmaps"]
  verbs: ["create"]
```

or a `ClusterRole` object

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: cluster-admin
rules:
- apiGroups:
  - '*'
  resources:
  - '*'
  verbs:
  - '*'
- nonResourceURLs:
  - '*'
  verbs:
  - '*'
```

You can add multiple rules in each `Role`.

Each rule has three sections, as we talked about earlier: `apiGroups`, `resources` and `verbs`.  
For the core group, we can leave the `apiGroups` section blank; for any other group, we need to specify it.

| Resource     | API group |
| ------------ | --------- |
| Pods         | `""`      |
| ConfigMaps   | `""`      |
| Services     | `""`      |
| Secrets      | `""`      |
| Deployments  | `apps`    |
| StatefulSets | `apps`    |
| DaemonSets   | `apps`    |
| Jobs         | `batch`   |
| CronJobs     | `batch`   |

When defining rules for `Roles`, you can optionally allow access to specific resources by adding a `resourceNames` field to the `rule`

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: developer
  namespace: default
rules:
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["get", "update", "delete"]
  resourceNames: ["blue", "orange"]
- apiGroups: [""]
  resources: ["configmaps"]
  verbs: ["create"]
```

> [!WARNING]
>
> `resourceNames` does not work with all the operations in the same way.  
> In particular:
>
> - for operations such as `list` and `watch`, the behavior requires careful attention because the request must specify the resource name using `fieldSelector`.
> - `resourceNames` cannot be used to restrict `create` operations to specific names, because a `create` request does not yet have a resource name available for authorization in the same way that operations on an existing resource do.

Create the `Role` or the `ClusterRole` with the usual command `kubectl create -f developer-role.yaml` or `kubectl create -f pod-reader-cluster-role.yaml`.

The next step is to link the user to that `Role`.  
In order to accomplish this, we define another object called `RoleBinding`

```yaml
kind: RoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: devuser-developer-binding
  namespace: default
subjects:
- kind: User
  name: dev-user
  apiGroup: rbac.authorization.k8s.io
- kind: ServiceAccount
  name: my-app
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: developer
  apiGroup: rbac.authorization.k8s.io
```

or `ClusterRoleBinding`

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: cluster-admin
subjects:
- kind: Group
  name: system:masters
  apiGroup: rbac.authorization.k8s.io
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
```

> [!NOTE]
>
> Remember that the subject `kind` in the binding can be one between:
>
> - User;
> - Group;
> - ServiceAccount.

We can create the object from the manifest with the usual command `kubectl create -f devuser-developer-binding.yaml` or `kubectl create -f dev-user-pod-reader.yaml`.

```text
┌─────────────┐
│    Role     │
│ permissions │
└──────┬──────┘
       │
       │ RoleBinding
       ▼
 ┌──────────┐
 │ Subject  │
 │ User     │
 │ Group    │
 │ SA       │
 └──────────┘
```

The `RoleBinding` or `ClusterRoleBinding` object associates one or more subjects (User, Group or ServiceAccount) to a `Role` or `ClusterRole` and, apart from the usual `kind`, `apiVersion` and `metadata` sections, has two main sections:

- `subjects` is where we specify the user details;
- `roleRef` is where we provide the details of the created `Role`.

> [!IMPORTANT]
>
> `Roles` and `RoleBindings` are namespaced resources.  
> In the examples above, `dev-user` gets access to `Pods` and `ConfigMaps` within the default namespace.  
> If you want to assign a different namespace instead of the default one, you can do so by specifying a different value in `metadata.namespace` in both `Roles` and `RoleBindings`.
>
> A `Role` cannot grant access to cluster-scoped resources like:
>
> - `Nodes`;
> - `PersistentVolumes`;
> - `Namespaces`;
> - `ClusterRoles`;
> - `ClusterRoleBindings`.
>
> To grant this type of access, you use a `ClusterRole`, which is typically bound using a `ClusterRoleBinding`.  
> For example, a `ClusterRole` that grants access to `nodes` is inherently useless when associated with a `RoleBinding`, because `nodes` do not belong to a namespace.
>
> Cluster-scoped resources → `ClusterRole` required.
> Namespace-scoped resources → `Role` or `ClusterRole`.

Keep in mind that the scope of the given permissions depends on the narrower of the scopes between the Binding and the Role

| Binding              | Role          | Scope of permissions |
| -------------------- | ------------- | -------------------- |
| `RoleBinding`        | `Role`        | namespace            |
| `RoleBinding`        | `ClusterRole` | namespace            |
| `ClusterRoleBinding` | `ClusterRole` | whole cluster        |

and it is not possible to bind a `ClusterRoleBinding` to a `Role`.

> [!IMPORTANT]
>
> The role type determines what can be referenced; the binding type determines where the permissions are applied.

In order to check if you have access to a particular resource in the cluster, you can use the command `kubectl auth can-i <verb> <resource>`, like for example

```console
$ kubectl auth can-i create deployments
yes

$ kubectl auth can-i delete nodes
no
```

If you are an administrator, you can even impersonate another user to check their permissions by adding the parameter `--as <username>`, for example

```console
$ kubectl auth can-i create deployments --as dev-user
no

$ kubectl auth can-i create pods --as dev-user
yes

$ kubectl auth can-i create pods --as dev-user --namespace test
no
```

Useful commands:

- `kubectl get roles [-n <namespace>]`: view the `Roles`;
- `kubectl get clusterroles`: view the `ClusterRoles`;
- `kubectl get rolebindings [-n <namespace>]`: view the `RoleBindings`;
- `kubectl get clusterrolebindings`: view the `ClusterRoleBindings`;
- `kubectl create role <role name> --verb=<verb1>[,<verb2>][,<verb3>] --resource=<resource>`: imperative command for creating a `Role`;
- `kubectl create clusterrole node-reader --verb=get,list,watch --resource=nodes`: imperative command example for creating a `ClusterRole`;
- `kubectl create rolebinding <rolebinding name> --role=<role name> --user=<user name>`: imperative command for creating a `RoleBinding`;
- `kubectl create rolebinding mike-node-reader --role=node-reader --user=mike`: imperative command example for creating a `RoleBinding`;
- `kubectl describe role <role_name>`: view the details of a specific `Role`;
- `kubectl describe rolebinding <rolebinding_name>`: view the details of a specific `RoleBinding`;
- `kubectl describe clusterrole node-reader`: view the details of a specific `ClusterRole` example;
- `kubectl describe clusterrolebinding mike-node-reader`: view the details of a specific `ClusterRoleBinding` example;
- `kubectl auth can-i <verb> <resource> [--as <username>]`: check if you can perform a specific action on a particular resource;
- `kubectl auth can-i --list [--as <username>]`: list the actions that can be performed on all resources.

---

### Admission Controllers

When a request hits the API Server, we know that it goes

1) first, through an authentication process;
2) then, through an authorization process (usually done through RBAC);
3) finally, through the admission control phase, where admission plugins evaluate the request;
4) if all the above steps are accomplished, the object of the request is persisted.

```text
         API Request
              │
              ▼
     ┌─────────────────┐
     │ Authentication  │
     │                 │
     │  Who are you?   │
     └────────┬────────┘
              │
              ▼
     ┌─────────────────┐
     │  Authorization  │
     │                 │
     │    What can     │
     │     you do?     │
     └────────┬────────┘
              │
              ▼
     ┌─────────────────┐
     │    Admission    │
     │                 │
     │ Is this request │
     │   acceptable?   │
     └────────┬────────┘
              │
     ┌────────┴────────┐
     │                 │
  Mutating         Validating
     │                 │
modify object       allow/deny
     │                 │
     └────────┬────────┘
              ▼
        Persistence (etcd)
```

An Admission Controller intercepts API requests after authentication and authorization and before the object is persisted. It can reject them, and if it is a mutating controller, it can also modify them.  
Admission controllers are perfect for filtering requests cluster-wide, regardless of the Users, Groups or ServiceAccounts they come from. For example:

- only permit images from specific registries (internal corporate registries instead of public ones like DockerHub);
- do not permit containers to run as the root user;
- only permit certain capabilities;
- enforce that pre-defined metadata and labels are declared in specific resources;
- automatically add labels or other metadata.

Admission controllers can be of type:

- **Validating**: checks specific configuration in order to accept or reject the request object;
- **Mutating**: modifies the object contained in the request before it is persisted.

> [!IMPORTANT]
>
> Mutating admission happens before validating admission.  
> In the first phase, only mutating admission plugins run. In the second phase, only validating admission plugins run.  
> This is so that any change made by the mutating admission controllers can be considered during the validation.

Admission controllers help us implement stronger security measures and enforce how a cluster is used.

Kubernetes provides several built-in admission plugins. Some are enabled by default, while others must be explicitly enabled.  
A non-exhaustive list of the most relevant is the following

| Admission controller         | Behavior                                                                                                                                                     | enabled by default |
| ---------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------ | ------------------ |
| `NamespaceLifecycle`         | Prevents operations on resources in namespaces that do not exist and handles requests involving namespaces that are terminating                              | :white_check_mark: |
| `LimitRanger`                | Enforces `LimitRange` constraints, such as default CPU/memory requests and limits and maximum/minimum resource limits for Pods and containers                | :white_check_mark: |
| `ResourceQuota`              | Enforces `ResourceQuota` limits for namespaces, restricting the amount/count of resources that can be created                                                | :white_check_mark: |
| `ServiceAccount`             | Automatically assigns the `default` ServiceAccount to Pods that do not specify one and can apply related ServiceAccount defaults                             | :white_check_mark: |
| `DefaultStorageClass`        | Observes creation of `PVC`s and automatically adds the default `StorageClass` if one is not specified                                                        | :white_check_mark: |
| `PodSecurity`                | Enforces the **Pod Security Standards** (`privileged`, `baseline`, `restricted`) on Pods, according to the policy configured for the namespace               | :white_check_mark: |
| `DefaultTolerationSeconds`   | Adds default tolerations for `not-ready` and `unreachable` taints, allowing Pods to remain bound to a node for a default amount of time before being evicted | :white_check_mark: |
| `AlwaysPullImages`           | Sets `imagePullPolicy` to `Always` in Pods so that container images are always pulled                                                                        | :x:                |
| `MutatingAdmissionWebhook`   | Calls external mutating webhooks that can modify the object before it is persisted                                                                           | :white_check_mark: |
| `ValidatingAdmissionWebhook` | Calls external validating webhooks that can accept or reject a request, but cannot modify the object                                                         | :white_check_mark: |
| `ValidatingAdmissionPolicy`  | Applies validation policies defined in Kubernetes using CEL expressions; can reject requests that do not satisfy the policy                                  | :white_check_mark: |

> [!NOTE]
>
> The exact default of enabled and disabled admission plugins highly depends on the Kubernetes version.
>
> There are also deprecated admission controllers, such as:
>
> - `EventRateLimit`: limits the rate at which Events can be accepted by the API server;
> - `NamespaceExists`: rejects requests targeting a namespace that does not exist;
> - `NamespaceAutoProvision`: automatically creates a namespace when a request targets a namespace that does not exist.
>
> The automatic namespace creation feature of the deprecated `NamespaceAutoProvision` controller has not been moved to a new built-in admission controller.  
> Instead, Kubernetes removed automatic provisioning behavior, and `NamespaceLifecycle` only rejects requests to missing namespaces.

To test the behavior of the `NamespaceLifecycle` admission controller we can do the following

```console
$ kubectl run nginx --image nginx --namespace blue
Error from server (NotFound): namespaces "blue" not found

$ kubectl get namespace
NAME              STATUS   AGE
default           Active   2d6h
kube-public       Active   2d6h
kube-system       Active   2d6h
```

You can **inspect the available admission plugin options and the default-enabled plugins** with

```bash
# From kubectl
kubectl exec -it kube-apiserver-controlplane -n kube-system -- kube-apiserver -h | grep admission-plugins

# Alternatively, if on the control-plane with the kube-apiserver
kube-apiserver -h | grep admission-plugins
```

> [!TIP]
>
> Replace `kube-apiserver-controlplane` with the name of the API Server `Pod` in your instance (for example `kube-apiserver-minikube` in Minikube)

The result will be something like that

```bash
      --admission-control strings                    Admission is divided into two phases. In the first phase, only mutating admission plugins run. In the second phase, only validating admission plugins run. The names in the below list may represent a validating plugin, a mutating plugin, or both. The order of plugins in which they are passed to this flag does not matter. Comma-delimited list of: AlwaysAdmit, AlwaysDeny, AlwaysPullImages, CertificateApproval, CertificateSigning, CertificateSubjectRestriction, ClusterTrustBundleAttest, DefaultIngressClass, DefaultStorageClass, DefaultTolerationSeconds, DenyServiceExternalIPs, EventRateLimit, ExtendedResourceToleration, ImagePolicyWebhook, LimitPodHardAntiAffinityTopology, LimitRanger, MutatingAdmissionPolicy, MutatingAdmissionWebhook, NamespaceAutoProvision, NamespaceExists, NamespaceLifecycle, NodeDeclaredFeatureValidator, NodeRestriction, OwnerReferencesPermissionEnforcement, PersistentVolumeClaimResize, PodNodeSelector, PodSecurity, PodTolerationRestriction, PodTopologyLabels, Priority, ResourceQuota, RuntimeClass, ServiceAccount, StorageObjectInUseProtection, TaintNodesByCondition, ValidatingAdmissionPolicy, ValidatingAdmissionWebhook. (DEPRECATED: Use --enable-admission-plugins or --disable-admission-plugins instead. Will be removed in a future version.)
      --disable-admission-plugins strings            admission plugins that should be disabled although they are in the default enabled plugins list (NamespaceLifecycle, LimitRanger, ServiceAccount, TaintNodesByCondition, PodSecurity, Priority, DefaultTolerationSeconds, DefaultStorageClass, StorageObjectInUseProtection, PersistentVolumeClaimResize, RuntimeClass, CertificateApproval, CertificateSigning, ClusterTrustBundleAttest, CertificateSubjectRestriction, DefaultIngressClass, PodTopologyLabels, NodeDeclaredFeatureValidator, MutatingAdmissionPolicy, MutatingAdmissionWebhook, ValidatingAdmissionPolicy, ValidatingAdmissionWebhook, ResourceQuota). Comma-delimited list of admission plugins: AlwaysAdmit, AlwaysDeny, AlwaysPullImages, CertificateApproval, CertificateSigning, CertificateSubjectRestriction, ClusterTrustBundleAttest, DefaultIngressClass, DefaultStorageClass, DefaultTolerationSeconds, DenyServiceExternalIPs, EventRateLimit, ExtendedResourceToleration, ImagePolicyWebhook, LimitPodHardAntiAffinityTopology, LimitRanger, MutatingAdmissionPolicy, MutatingAdmissionWebhook, NamespaceAutoProvision, NamespaceExists, NamespaceLifecycle, NodeDeclaredFeatureValidator, NodeRestriction, OwnerReferencesPermissionEnforcement, PersistentVolumeClaimResize, PodNodeSelector, PodSecurity, PodTolerationRestriction, PodTopologyLabels, Priority, ResourceQuota, RuntimeClass, ServiceAccount, StorageObjectInUseProtection, TaintNodesByCondition, ValidatingAdmissionPolicy, ValidatingAdmissionWebhook. The order of plugins in this flag does not matter.
      --enable-admission-plugins strings             admission plugins that should be enabled in addition to default enabled ones (NamespaceLifecycle, LimitRanger, ServiceAccount, TaintNodesByCondition, PodSecurity, Priority, DefaultTolerationSeconds, DefaultStorageClass, StorageObjectInUseProtection, PersistentVolumeClaimResize, RuntimeClass, CertificateApproval, CertificateSigning, ClusterTrustBundleAttest, CertificateSubjectRestriction, DefaultIngressClass, PodTopologyLabels, NodeDeclaredFeatureValidator, MutatingAdmissionPolicy, MutatingAdmissionWebhook, ValidatingAdmissionPolicy, ValidatingAdmissionWebhook, ResourceQuota). Comma-delimited list of admission plugins: AlwaysAdmit, AlwaysDeny, AlwaysPullImages, CertificateApproval, CertificateSigning, CertificateSubjectRestriction, ClusterTrustBundleAttest, DefaultIngressClass, DefaultStorageClass, DefaultTolerationSeconds, DenyServiceExternalIPs, EventRateLimit, ExtendedResourceToleration, ImagePolicyWebhook, LimitPodHardAntiAffinityTopology, LimitRanger, MutatingAdmissionPolicy, MutatingAdmissionWebhook, NamespaceAutoProvision, NamespaceExists, NamespaceLifecycle, NodeDeclaredFeatureValidator, NodeRestriction, OwnerReferencesPermissionEnforcement, PersistentVolumeClaimResize, PodNodeSelector, PodSecurity, PodTolerationRestriction, PodTopologyLabels, Priority, ResourceQuota, RuntimeClass, ServiceAccount, StorageObjectInUseProtection, TaintNodesByCondition, ValidatingAdmissionPolicy, ValidatingAdmissionWebhook. The order of plugins in this flag does not matter.
```

> [!IMPORTANT]
>
> `--admission-control` is deprecated in favor of `--enable-admission-plugins` and `--disable-admission-plugins`

To find out, instead, **which plugins are configured manually on your API server**, you can:

```bash
# Inspect the manifest of the static pod
kubectl get pod kube-apiserver-controlplane -n kube-system -o yaml | grep -E "enable-admission-plugins|disable-admission-plugins"

# Alternatively, if on the control-plane with the kube-apiserver
cat /etc/kubernetes/manifests/kube-apiserver.yaml | grep -E "enable-admission-plugins|disable-admission-plugins"
# Or
grep admission /etc/kubernetes/manifests/kube-apiserver.yaml
```

To **enable or disable admission controllers**, you need to edit the kube-apiserver configuration file

```bash
vi /etc/kubernetes/manifests/kube-apiserver.yaml
```

and then modify `--enable-admission-plugins` or `--disable-admission-plugins` accordingly.

There are also external **Admission Webhooks**, which are external services that Kubernetes can call in order to make an admission decision for a request.

You can build and deploy a webhook server by yourself, as a standalone server or as a `Deployment` in your Kubernetes cluster.  
Keep in mind that many modern admission controllers are implemented as external extensions rather than built directly into the Kubernetes API server.  
Examples include policy engines such as Kyverno and OPA Gatekeeper.

Kubernetes provides two built-in admission plugins specifically for invoking external webhooks:

- `MutatingAdmissionWebhook`: invokes external webhooks that can modify the object;
- `ValidatingAdmissionWebhook`: invokes external webhooks that can accept or reject the request, but cannot modify the object.

> [!NOTE]
>
> `MutatingAdmissionWebhook` and `ValidatingAdmissionWebhook` are admission controllers themselves.

The webhooks are configured using Kubernetes API resources:

- `MutatingWebhookConfiguration`: used when the webhook needs to modify the object;
- `ValidatingWebhookConfiguration`: used when the webhook only needs to validate the request.

You can list the configured webhooks using the standard `kubectl` commands:

```bash
# List the configured validating webhooks
kubectl get validatingwebhookconfigurations

# List the configured mutating webhooks
kubectl get mutatingwebhookconfigurations
```

You can inspect a specific configuration with:

```bash
kubectl describe validatingwebhookconfiguration <name>

kubectl describe mutatingwebhookconfiguration <name>
```

or retrieve the complete configuration in YAML format:

```bash
kubectl get validatingwebhookconfiguration <name> -o yaml

kubectl get mutatingwebhookconfiguration <name> -o yaml
```

An Admission Webhook is typically exposed as an HTTPS endpoint running inside or outside the Kubernetes cluster.  
It works this way:

- the API server sends to the webhook the `AdmissionReview` object in a JSON format;

  ```json
  {
    "apiVersion": "admission.k8s.io/v1",
    "kind": "AdmissionReview",
    "request": {
      "uid": "705ab4f5-6393-11e8-b7a1-42010a800002",
      "kind": {
        "group": "",
        "version": "v1",
        "kind": "Pod"
      },
      "resource": {
        "group": "",
        "version": "v1",
        "resource": "pods"
      },
      "requestKind": {
        "group": "",
        "version": "v1",
        "kind": "Pod"
      },
      "requestResource": {
        "group": "",
        "version": "v1",
        "resource": "pods"
      },
      "name": "nginx-pod",
      "namespace": "default",
      "operation": "CREATE",
      "userInfo": {
        "username": "kubernetes-admin",
        "groups": [
          "system:masters",
          "system:authenticated"
        ]
      },
      "object": {
        "apiVersion": "v1",
        "kind": "Pod",
        "metadata": {
          "name": "nginx-pod",
          "namespace": "default"
        },
        "spec": {
          "containers": [
            {
              "name": "nginx",
              "image": "nginx:latest"
            }
          ]
        }
      },
      "oldObject": null,
      "dryRun": false,
      "options": {
        "apiVersion": "meta.k8s.io/v1",
        "kind": "CreateOptions"
      }
    }
  }
  ```

- the webhook evaluates the request according to its own policies and returns an `AdmissionReview` response to the API server.  
  For example, we can have an accepted request from the validating webhook

  ```json
  {
    "apiVersion": "admission.k8s.io/v1",
    "kind": "AdmissionReview",
    "response": {
      "uid": "705ab4f5-6393-11e8-b7a1-42010a800002",
      "allowed": true
    }
  }
  ```

  or a denied one

  ```json
  {
    "apiVersion": "admission.k8s.io/v1",
    "kind": "AdmissionReview",
    "response": {
      "uid": "705ab4f5-6393-11e8-b7a1-42010a800002",
      "allowed": false,
      "status": {
        "code": 403,
        "message": "The use of the tag 'latest' is not allowed for images in this cluster."
      }
    }
  }
  ```

  and in case of mutating webhook, we can have a patch to apply to the object

  ```json
  {
    "apiVersion": "admission.k8s.io/v1",
    "kind": "AdmissionReview",
    "response": {
      "uid": "705ab4f5-6393-11e8-b7a1-42010a800002",
      "allowed": true,
      "patchType": "JSONPatch",
      "patch": "W3sib3AiOiJhZGQiLCJwYXRoIjoiL21ldGFkYXRhL2xhYmVscyIsInZhbHVlIjp7ImVudiI6InByb2R1Y3Rpb24ifX1d"
    }
  }
  ```

For a validating webhook, the `AdmissionReview.AdmissionResponse.allowed` field of the response contains a decision such as:

- `true`: the request is accepted;
- `false`: the request is rejected.

A mutating webhook can additionally return a set of modifications to apply to the object, typically using a JSON Patch.

This is a simplified representation of the webhooks flow (the actual admission chain can contain multiple built-in plugins and multiple webhooks):

```text
      Kubernetes API Server
              │
              │ Admission request
              ▼
┌────────────────────────────┐
│  MutatingAdmissionWebhook  │
└─────────────┬──────────────┘
              │
              ▼
      External Webhook
              │
  modified or default object
              │
              ▼
┌────────────────────────────┐
│ ValidatingAdmissionWebhook │
└─────────────┬──────────────┘
              │
              ▼
      External Webhook
              │
        allow or deny
              │
              ▼
            etcd
```

The webhook configuration specifies which requests should be sent to the external service.  
For example, a webhook can be configured to intercept:

- only CREATE requests;
- only UPDATE requests;
- only Pods;
- only resources belonging to a specific API group;
- only requests targeting specific namespaces;
- a combination of these conditions.

This allows a webhook to apply a policy only to the resources and operations for which it is relevant.

A simplified `ValidatingWebhookConfiguration` looks like this:

```yaml
apiVersion: admissionregistration.k8s.io/v1
kind: ValidatingWebhookConfiguration
metadata:
  name: example-validator
webhooks:
- name: validator.example.com
  rules:
  - apiGroups: [""]
    apiVersions: ["v1"]
    operations: ["CREATE", "UPDATE"]
    resources: ["pods"]
  clientConfig:
    service:
      namespace: webhook-system
      name: validator
      path: /validate
    caBundle: <CA_BUNDLE>
  admissionReviewVersions: ["v1"]
  sideEffects: None
```

The important fields are:

- `rules`: defines which API requests are intercepted;
- `clientConfig`: specifies where the webhook is located, via:
  - `service`: identifies the Kubernetes Service exposing the webhook;
  - `url`: if deploying the server externally and it is not part of a Kubernetes cluster, used instead of `service`, specifying the exact address of that server;
  - `path`: HTTP path invoked by the API server;
- `caBundle`: CA certificate used by the API server to verify the webhook server certificate;
- `admissionReviewVersions`: specifies the `AdmissionReview` API versions supported by the webhook;
- `sideEffects`: declares whether calling the webhook has side effects.

The API server communicates with the webhook using the `AdmissionReview` API.

The webhook then returns an `AdmissionReview` response containing the admission decision.

> [!IMPORTANT]
>
> An Admission Webhook is not an authentication or authorization mechanism.  
> Authentication answers "Who are you?".  
> Authorization answers "Are you allowed to perform this operation?".  
> Admission answers "Should this request/object be accepted?".  
> A webhook is simply an external component that participates in the admission phase.

Other relevant fields of Admission Webhooks are:

- `failurePolicy`: specifies what the API server should do if the webhook cannot be reached or fails to respond successfully:
  - `Fail`: the request is denied;
  - `Ignore`: the request continues without the decision of the webhook;
- `namespaceSelector`: allows the webhook to be triggered only for namespaces that match specific labels;
- `objectSelector`: allows the webhook to be triggered only for objects that match specific labels.

```yaml
apiVersion: admissionregistration.k8s.io/v1
kind: MutatingWebhookConfiguration
webhooks:
  - name: my-webhook.example.com
    objectSelector:
      matchLabels:
        foo: bar
    namespaceSelector:
      matchExpressions:
        - key: runlevel
          operator: NotIn
          values: ["0","1"]
    rules:
      - operations: ["CREATE"]
        apiGroups: ["*"]
        apiVersions: ["*"]
        resources: ["*"]
```

---

### API versions

When an API version is at `v1`, it means that it is a Generally Available stable version.  
But there are other version kinds, like the following:

| Binding            | Alpha                                            | Beta                                                 | GA (stable)                             |
| ------------------ | ------------------------------------------------ | ---------------------------------------------------- | --------------------------------------- |
| Version Name       | vXalphaY (eg: `v1alpha1`)                        | vXbetaY (eg: `v1beta1`)                              | vX (eg: `v1`)                           |
| Enabled by default | No. Can be enabled via flags                     | Usually no ¹                                         | Yes by default                          |
| Tests              | May lack e2e tests                               | Has e2e tests                                        | Has conformance tests                   |
| Reliability        | May have bugs                                    | May have minor bugs                                  | Highly reliable                         |
| Support            | No commitment; may be dropped later              | Commits to complete the feature and move to GA       | Will be present in many future releases |
| Audience           | Expert users interested in giving early feedback | Users interested beta testing and providing feedback | All users                               |

¹ Beta APIs introduced before Kubernetes 1.22 were normally enabled by default.

Alpha versions are used when an API is first developed and becomes part of the Kubernetes release for the very first time.  
Alpha API groups are not enabled by default.

An API may graduate from Alpha to Beta and eventually to GA when the required stability, testing and design criteria are met.

The current API Groups with their served versions are documented here <https://kubernetes.io/docs/reference/kubernetes-api/group-versions/>.  
If you look at this page you'll notice that some of those APIs have multiple versions.  
This is because an API group can support multiple versions at the same time, allowing users to use the desired version based on stability / innovativeness requirements.  
But even though multiple versions can coexist, only one can be the preferred version and only one can be the storage version.

![API groups](./images/02-api-groups.png "API groups")

> [!NOTE]
>
> The example above is for illustrative purposes only.
> It does not reflect the real version history of the `Deployment` API.
> The `apps` API group currently only serves the `v1` version.

The preferred version is the one used if not differently specified (like when performing the commands `kubectl get <resource>` or `kubectl explain <resource>`).  
The storage version is the version used by the API server to serialize objects when they are persisted in the storage backend (usually etcd).  
The API server transparently converts between the API version used by the client and the version used for storage.  

```text
Client
  ↓
API version requested
  ↓
kube-apiserver
  ↓
conversion
  ↓
storage version
  ↓
etcd
```

You can check the preferred version with a command like the following

```bash
kubectl get --raw "/apis/batch" | jq
```

```json
{
  "kind": "APIGroup",
  "apiVersion": "v1",
  "name": "batch",
  "versions": [
    {
      "groupVersion": "batch/v1",
      "version": "v1"
    }
  ],
  "preferredVersion": {
    "groupVersion": "batch/v1",
    "version": "v1"
  }
}
```

> [!NOTE]
>
> (Advanced / useful for understanding, not normally needed for CKAD)
>
> As of now, it is not possible to see the storage version of a particular API through an API command.  
> One way to find it out is to look at the stored object in the etcd server itself.
>
> ```console
> $ ETCDCTL_API=3 etcdctl \
>   --endpoints=https://:2379 \
>   --cacert=/etc/kubernetes/pki/etcd/ca.crt \
>   --cert=/etc/kubernetes/pki/etcd/server.crt \
>   --key=/etc/kubernetes/pki/etcd/server.key \
>   get "/registry/deployments/default/blue" --print-value-only
>
> k8s
>
> apps/v1
> Deployment
>
>
> bluedefault" * $cf8dcd55-8819-4be2-85e7-bb71665c2ddf2ZB
> successfully progresse8"2
> ```

To enable or disable a specific version, we must add it to the `--runtime-config` parameter of the kube-apiserver service:

- with no value or `=true` if we want to enable a specific API group;
- with the `=false` value if we want to disable a specific API group.

```diff
ExecStart=/usr/local/bin/kube-apiserver \\
  --advertise-address=\${INTERNAL_IP} \\
  --allow-privileged=true \\
  --apiserver-count=3 \\
  --authorization-mode=Node,RBAC \\
  --bind-address=0.0.0.0 \\
  --enable-swagger-ui=true \\
  --etcd-cafile=/var/lib/kubernetes/ca.pem \\
  --etcd-certfile=/var/lib/kubernetes/apiserver-etcd-client.crt \\
  --etcd-keyfile=/var/lib/kubernetes/apiserver-etcd-client.key \\
  --etcd-servers=https://127.0.0.1:2379 \\
  --event-ttl=1h \\
  --kubelet-certificate-authority=/var/lib/kubernetes/ca.pem \\
  --kubelet-client-certificate=/var/lib/kubernetes/apiserver-etcd-client.crt \\
  --kubelet-client-key=/var/lib/kubernetes/apiserver-etcd-client.key \\
  --kubelet-https=true \\
+ --runtime-config=batch/v2alpha1,authentication.k8s.io/v1beta1=true,storage.k8s.io/v1beta1=false \\
  --service-account-key-file=/var/lib/kubernetes/service-account.pem \\
  --service-cluster-ip-range=10.32.0.0/24 \\
  --service-node-port-range=30000-32767 \\
  --client-ca-file=/var/lib/kubernetes/ca.pem \\
  --tls-cert-file=/var/lib/kubernetes/apiserver.crt \\
  --tls-private-key-file=/var/lib/kubernetes/apiserver.key \\
  --v=2
```

After having modified `--runtime-config`, you need to restart the kube-apiserver (and the controller manager) to apply the changes.

There are also the following options:

- `api/all=false`: disables all API versions that are not strictly necessary (use with caution by combining it with exceptions, e.g., `api/all=false,api/v1=true`);
- `api/beta=false`: bulk-disables all Beta APIs across the cluster;
- `api/alpha=false`: bulk-disables all Alpha APIs across the cluster.

that you can combine in interesting ways, like `--runtime-config=api/all=false,api/v1=true` which disables all the APIs and then re-enables only `v1`.

---

### API deprecations

Deprecated and removed APIs are different concepts.

```text
Deprecated: API continues to work
    ↓
Warning: migration should be planned
    ↓
Removed: API no longer served
```

Starting from Kubernetes 1.19, when a REST API is deprecated, the API server:

- returns an HTTP Warning header;
- adds `k8s.io/deprecated: "true"` to the audit event;
- exposes the metric `apiserver_requested_deprecated_apis`.

API deprecation follows [several rules](https://kubernetes.io/docs/reference/using-api/deprecation-policy/).

**Rule #1**  
API elements may only be removed by incrementing the version of the API group.

For example, if an API element exists in `v1alpha1`, it cannot simply be removed from `v1alpha1`. A new API version such as `v1alpha2` must be introduced.

![CRD deprecation policy](./images/03-crd-deprecation-policy.png "CRD deprecation policy")

**Rule #2**  
API objects must be able to round-trip between API versions in a given release without information loss, with the exception of whole REST resources that do not exist in some versions.

If a resource is converted from `v1alpha1` to `v1alpha2`, even if the structure of the resource in the `v1alpha2` is different from the one in the `v1alpha1` (for example an extra field is present), when converting that resource from `v1alpha2` to `v1alpha1`, the result should be identical to the one before the first conversion.

![CRD round trip](./images/04-crd-round-trip.png "CRD round trip")

Furthermore, if `v1alpha2` introduces a new field, it must be possible to represent it even when performing a round-trip through `v1alpha1`, possibly using an equivalent field or an annotation.

**Rule #3a**  
An API version cannot be deprecated in favor of a less stable version.

GA → Beta       :x:
GA → Alpha      :x:
Beta → Alpha    :x:
Alpha → Beta    :white_check_mark:
Beta → GA       :white_check_mark:
Alpha → GA      :white_check_mark:

An API version may only be deprecated in favor of an API version with the same or higher stability level.

**Rule #4a**  
Other than the most recent API versions in each track, older API versions must
be supported after their announced deprecation for a duration of no less than:

- **GA**: must be supported for at least 12 months or 3 releases after deprecation, whichever is longer; after deprecation, must not be removed within the same major version;
- **Beta**: must be supported for at least 9 months or 3 minor releases after introduction, whichever is longer; after deprecation, it must remain served for at least 9 months or 3 minor releases, whichever is longer;
- **Alpha**: can be removed in any release without previous deprecation notice.

**Rule #4b**  
The "preferred" API version and the "storage" version for a given group may not advance until after a release has been made that supports both the new version and the previous version.

This allows an upgrade to be rolled back while maintaining compatibility with objects stored using the previous API/storage version.

The `kubectl convert` command lets you convert YAML manifests with the syntax `kubectl convert -f <old file> --output-version <new-api>`.  
For example `kubectl convert -f nginx.yaml --output-version apps/v1`.  
The `kubectl convert` command may not be available on the system because it is a plugin that needs to be installed separately.

---

## Custom resources and controllers

---

### Controllers & reconciliation

The behavior of the resources (seen until now) is managed by their respective controller.  
Many Kubernetes resources are managed by controllers.  
A controller continuously watches one or more resources and takes actions to make the actual state match the desired state.

> [!NOTE]
>
> Kubernetes does not necessarily have a single controller for every resource.  
> For example:
>
> - the `Deployment` controller creates and manages the `ReplicaSet`, while the `ReplicaSet` controller ensures that the desired number of `Pods` exists;
>
>   ```text
>   Deployment
>       │
>       │ Deployment controller
>       ▼
>   ReplicaSet
>       │
>       │ ReplicaSet controller
>       ▼
>   Pods
>   ```
>
> - for the `Service`, there isn't a single dedicated "Service controller" that spawns further child resources the way the `Deployment`/`ReplicaSet` controllers do; its behavior is split across several components (the endpoints/EndpointSlice controllers, `kube-proxy`, and — only for `LoadBalancer`-type Services — the "service controller" inside the cloud-controller-manager).

The controller is a process that runs in the background and continuously monitors the status of resources it is supposed to manage.  
When we create or update a resource, the desired state is persisted by the API server in etcd. Controllers continuously observe the resources through the Kubernetes API and reconcile the actual state with the desired state.  
When a resource is deleted, the API server removes it from etcd immediately, unless the resource uses finalizers — in that case, the deletion request is recorded (via a deletion timestamp) and the corresponding controllers must remove their finalizers before the object is actually deleted.

---

### CustomResourceDefinitions (CRDs)

Custom resources are an extension of the Kubernetes API that is not necessarily available in a default Kubernetes installation.

```text
             CRD
              │
              │ defines
              ▼
       Custom Resource
              │
              │ stored through
              ▼
     Kubernetes API Server
              │
              ├──────────────► etcd
              │
              │ watched by
              ▼
      Custom Controller
              │
              │ reconciliation
              ▼
      Desired → Actual
              │
      ┌───────┴────────┐
      ▼                ▼
Kubernetes          External
resources           systems
```

`CustomResourceDefinition`s (`CRD`s) extend the Kubernetes API.  
Custom resources are objects stored and managed through that API.  

---

### Custom controllers

A custom controller gives those objects a behavior by continuously reconciling their desired state with their actual state.

```text
    ┌──────────────────┐
    │   Kubernetes     │
    │    API Server    │
    └────────┬─────────┘
             │
  ┌──────────┴──────────┐
  ▼                     ▼
etcd              Custom Controller
                        │
                        │ reconciliation
                        ▼
                  External system
```

Controllers are usually written in Go.  
Here is an example of the [Deployment controller](https://github.com/kubernetes/kubernetes/blob/master/pkg/controller/deployment/deployment_controller.go).

Once the CRD has been created, the Kubernetes API server exposes the new resource type and we can perform CRUD operations on its instances. Create and update requests are validated against the CRD's OpenAPI schema. Invalid objects are rejected by the API server.  

If we want the resource to actually have a behavior, we also need to take care of a **custom controller**.

> [!TIP]
>
> A custom resource does not require a custom controller.  
> A controller is needed when we want the custom resource to trigger some additional behavior or reconciliation logic.

---

### An example of custom resources, CRDs, and custom controllers

For example, we can make a custom resource called `FlightTicket` whose controller watches the resource and, when its desired state changes, contacts an external API to perform the actual ticket booking actions.

![CRD FlightTicket](./images/05-crd-FlightTicket.png "CRD FlightTicket")

But in order to use a custom resource, we have to first **define** it.  
For that, we need a `CustomResourceDefinition`

```yaml
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata:
  name: flighttickets.flights.com
spec:
  scope: Namespaced # Either Namespaced or Cluster
  group: flights.com # The API group used in the resource's apiVersion
  names: # Names of the resource
    plural: flighttickets # To be used in the URL: /apis/<group>/<version>/<plural>
    singular: flightticket # To be used as an alias on the CLI and for display
    kind: FlightTicket # Normally the CamelCased singular type. Your resource manifests use this
    shortNames: # Allow shorter string to match your resource on the CLI
    - ft
  versions: # List of versions supported by this CustomResourceDefinition
    - name: v1
      served: true # Each version can be enabled/disabled by this Served flag
      storage: true # One and only one version must be marked as the storage version
      schema: # Defines all the parameters (fields and type of values) that can be specified under the spec section
        openAPIV3Schema:
          type: object
          properties:
            spec:
              type: object
              properties:
                from:
                  type: string
                to:
                  type: string
                number:
                  type: integer
                  minimum: 1 # We can also specify validation values like minimum and maximum
                  maximum: 10
              required:
                - from
                - to
                - number
```

> [!NOTE]
>
> The scope field determines whether instances of the custom resource are namespaced or cluster-scoped.  
> Custom Resource instances can be Namespaced or Cluster-scoped but **the CRD itself is always cluster-scoped**.

We can now create the `CRD` with the usual command

```console
$ kubectl create -f flightticket-custom-definition.yaml
customresourcedefinition.apiextensions.k8s.io/flighttickets.flights.com created
```

You can also get details of the created `CRD` with the following commands

```bash
kubectl explain flightticket
kubectl explain flightticket.spec
kubectl api-resources | grep flight
kubectl get crd
```

and manage its resources

```yaml
apiVersion: flights.com/v1
kind: FlightTicket
metadata:
  name: my-flight-ticket
spec:
  from: Mumbai
  to: London
  number: 2
```

```console
$ kubectl create -f flightticket.yaml
flightticket.flights.com/my-flight-ticket created

$ kubectl get ft
NAME               AGE
my-flight-ticket   18s

$ kubectl delete -f flightticket.yaml
flightticket.flights.com "my-flight-ticket" deleted
```

That solves the first part of our problem: being able to create and manage any kind of object type that we want on Kubernetes.  
But this only allows us to create, manage and store these objects through the Kubernetes API (which then stores them in etcd). Without a controller, Kubernetes does not know what additional actions should be performed (for example, when a FlightTicket is created or modified).  
The second part is building a controller that can watch when these resources are created and perform actions based on the resource's specification.

```text
User
  │
  │ create/update FlightTicket
  ▼
API Server
  │
  ├──────────────► etcd
  │
  ▼
Custom Controller
  │
  │ watches API
  ▼
External API
```

> [!NOTE]
>
> The resource controller should not read from etcd directly.  
> It reads objects through the Kubernetes API server (typically via client-go/controller-runtime), even though the object is physically stored in etcd.  
> The API server is the abstraction layer through which Kubernetes resources are accessed.

Now we need to monitor the status of the objects and perform the actions on the corresponding external services, through the custom controller.  
A controller is any process or code that runs in a loop and continuously monitors the Kubernetes cluster, listening to events of specific objects being changed and performing the appropriate actions in response.

> [!TIP]
>
> Controllers are commonly written in Go and can use Kubernetes libraries such as client-go and controller frameworks such as controller-runtime.  
> Developing a controller in a different language may be challenging: without an equivalent to client-go's informers, you don't get built-in caching and queuing, so direct calls to the API can become expensive and you would need to build your own frameworks to manage them.

The Kubernetes [sample-controller](https://github.com/kubernetes/sample-controller) repository provides an example of how a custom controller can be implemented.  
In a real project, the controller would also require the appropriate API types, clients, RBAC configuration and deployment.  
Here follows an example of `controller.go` with our custom logic.

> [!NOTE]
>
> Controller development is beyond the CKAD scope. The following is included only to illustrate how a controller implements the reconciliation logic.

```Go
package flightticket

var controllerKind =
    flightsv1.SchemeGroupVersion.WithKind("FlightTicket")

// < Code hidden >

// Run begins watching and syncing.
func (dc *FlightTicketController) Run(workers int,
    stopCh <-chan struct{}) {
    // ...
}

// < Code hidden >

// Call BookFlightAPI
func (dc *FlightTicketController) callBookFlightAPI(obj interface{}) {
    // ...
}

// < A lot of code hidden >
```

We can then build our custom controller

```console
$ go build -o sample-controller .
go: downloading k8s.io/client-go v0.0.0-20211001003700-dbfa30b9d908
go: downloading golang.org/x/text v0.3.6
```

and run it specifying the `kubeconfig` file, which the controller can use to authenticate to the Kubernetes API.

```console
$ ./sample-controller -kubeconfig=$HOME/.kube/config
I1013 02:11:07.489479 40117 controller.go:115] Setting up event handlers
I1013 02:11:07.489701 40117 controller.go:156] Starting FlightTicket controller
```

Once the controller is ready, we can decide how to distribute it.  
We don't want to build and run it each time; instead, we may package the controller in a Docker image, and then run it inside the cluster as a `Pod` or a `Deployment`.

---

### Operators

We've seen until now the **CRD** and the **Custom Controller**, which are two separate entities that are necessary to make use of **custom resources** with a behavior.  
An **Operator** is a software application that extends Kubernetes with custom resources and controllers, combining them to automate the management of a specific application or system.

```text
Operator
│
├── CRDs
│
├── Controllers
│
└── Application-specific automation logic
```

An Operator may install the required `CRDs` and run its controllers inside the cluster, often as a `Deployment`.  
But an Operator can do much more than just deploying those two components.  
For example, one of the most popular operators is the etcd operator, which provides the following CRDs, controllers and capabilities:

| CRD         | custom controller  | capability                        |
| ----------- | ------------------ | --------------------------------- |
| EtcdCluster | Etcd controller    | deploy and manage an etcd cluster |
| EtcdBackup  | Backup controller  | manage backups                    |
| EtcdRestore | Restore controller | manage restore operations         |

And you can manage the above operations by simply creating a custom resource.

Kubernetes operators typically do what a human operator would do to manage a specific application (installing it, maintaining it, taking backups and restoring them when needed).

Many operators are available on [OperatorHub](https://operatorhub.io/), where you can find operators for many popular applications.

> [!TIP]
>
> If there isn't an operator in the ecosystem that implements the desired behavior, it's possible to [create a customized one](https://kubernetes.io/docs/concepts/extend-kubernetes/operator/#writing-operator).  
> There are few libraries and tools that can be used to write cloud native operator, like [Operator Framework](https://operatorframework.io/).

---

## Helm Fundamentals

Kubernetes is really powerful in managing complex infrastructures at scale, but we humans tend to struggle with this complexity.

For example, even a relatively simple WordPress site might need the following:

- a `Deployment` with the application;
- a `PersistentVolume` and a `PersistentVolumeClaim` to store the database;
- a `Service` to expose the web server;
- a `Secret` to store the admin password;
- and more if you want extra capabilities.

```text
                        / \
                       /   \                       + - - - - - - +
                      /  ^  \                      '             '
                     /       \                     '     PVC     '
                    / Service \                    '             '
                   /___________\                   + - - - - - - +

      +---------------------------------------+    +-------------+
      |              Deployment               |    |             |
      |                                       |    |     PV      |
      |   ( * )        ( * )        ( * )     |    |             |
      |   Pod 1        Pod 2        Pod 3     |    +-------------+
      +---------------------------------------+

        .---------.              +------------------+
       /  (  W  )  \             |                  |
      |  WordPress  |            |      Secret      |
       \           /             |                  |
        '---------'              +------------------+
```

For every object, we might need a separate YAML file: `deployment.yaml`, `pv.yaml`, `pvc.yaml`, `service.yaml` and `secret.yaml`.  
Then, at application creation, we need to run `kubectl apply` on every YAML file.  
We also need to identify all the manifests concerned whenever a change is needed, carefully modify them, and then apply the changes.  
Finally, when we delete the application, we need to `kubectl delete` all of them.

> [!CAUTION]
>
> You could use a single YAML file for all the resources instead of having one for each, but it will probably make it even harder to identify which parts are responsible for the required resources.

Helm changes this paradigm.
If Kubernetes takes care of making all the individually declared objects exist in our cluster, Helm takes care of the representation of an application as a whole.  
Helm looks at those objects as part of one big package, and that's why it's also defined as a package manager for Kubernetes.  
We don't tell Helm which objects it should touch, we just tell it instead what package we want it to act on.

With Helm, instead of having multiple manifests in multiple YAML files, you can specify all the (default) values in the `values.yaml` file of the chart.  

```yaml
## User of the application
## ref: https://github.com/bitnami/charts/blob/main/bitnami/wordpress/values.yaml
##
wordpressUsername: user

## Application password
## Defaults to a random 10-character alphanumeric string...
## ref: https://github.com/bitnami/charts/blob/main/bitnami/wordpress/values.yaml
##
# wordpressPassword:

## Admin email
## ref: https://github.com/bitnami/charts/blob/main/bitnami/wordpress/values.yaml
##
wordpressEmail: user@example.com

## First name
## ref: https://github.com/bitnami/charts/blob/main/bitnami/wordpress/values.yaml
##
wordpressFirstName: FirstName

## Last name
## ref: https://github.com/bitnami/charts/blob/main/bitnami/wordpress/values.yaml
##
wordpressLastName: LastName
```

The first step is to convert relevant values from the manifest files to variables.  

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: wordpress
  labels:
    app: wordpress
spec:
  selector:
    matchLabels:
      app: wordpress
      tier: frontend
  strategy:
    type: Recreate
  template:
    metadata:
      labels:
        app: wordpress
        tier: frontend
    spec:
      containers:
      - image: {{ .Values.image }}
        name: wordpress
```

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: wordpress-pv
spec:
  capacity:
    storage: {{ .Values.storage }}
  accessModes:
  - ReadWriteOnce
  gcePersistentDisk:
    pdName: wordpress-2
    fsType: ext4
```

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: wp-pv-claim
  labels:
    app: wordpress
spec:
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: {{ .Values.storage }}
```

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: wordpress-admin-password
data:
  key: {{ .Values.passwordEncoded }}
```

Then, those variable values are defined in the `values.yaml` file, for example:

```yaml
image: wordpress:4.8-apache
storage: 30Gi
passwordEncoded: SGVsbG8sIEFsZXNzYW5kcm8ncyBub3RlcyBoZXJl
```

> [!WARNING]
>
> Values files are not a secure secret store.  
> Sensitive values placed in values.yaml can be exposed in source control or Helm configuration.

Finally, you can install it with:

```bash
helm install wordpress ./wordpress
```

> [!NOTE]
>
> A chart can be installed from a repository, a local directory, an archive, and also directly from an OCI registry.  
> For example, all the following are valid:
>
> - `helm install wordpress bitnami/wordpress`
> - `helm install wordpress ./wordpress`
> - `helm install wordpress https://example.com/wordpress-1.0.0.tgz`
> - `helm install wordpress oci://registry-1.docker.io/bitnamicharts/wordpress`
>
> The same applies to the `helm upgrade` command.

Furthermore, you can set:

- custom settings in another specific file that you can give as input, with the `-f` option

  ```bash
  helm install my-release ./mychart -f my-values.yaml
  ```

- custom settings via specific variables, with the `--set` option

  ```bash
  helm install my-release ./mychart --set image.tag=1.2.3
  ```

> [!TIP]
>
> Keep in mind that:
>
> - the `values.yaml` of the chart represents the default configuration;
> - the values passed with the `-f` parameter take precedence over the default values and, if present, override them;
> - the values passed explicitly with the `--set` option take precedence over the previous two and, if present, override them both.

So a combination of templates + the values gives us the final version of definition files that can be used to deploy the application.  
Together, the templates, values, and chart metadata form a Helm chart.

```text
         CHART
           │
    +      │
    │      │
  VALUES   │
    │      │
    └───┬──┘
        ▼
Template rendering
        │
        ▼
Kubernetes manifests
        │
        ▼
  Kubernetes API
        │
        ▼
       etcd
```

A single Helm chart may be used to deploy a simple application and has:

- the template files;
- the values.yaml file with the variables;
- `Chart.yaml` with information about the chart itself.

We can create our own chart for the application or explore the existing charts from [Artifact Hub](https://artifacthub.io/) uploaded from other users.  
You can search for them either using the web interface or the CLI

```console
$ helm search hub wordpress
URL                                                     CHART VERSION   APP VERSION             DESCRIPTION
https://artifacthub.io/packages/helm/slybase-wo...      5.5.3           7.0.1                   Using the official WordPress image. This chart ...
https://artifacthub.io/packages/helm/wordpress-...      1.0.7           7.0.4                   WordPress is the world's most popular blogging ...
https://artifacthub.io/packages/helm/quench-wor...      0.0.10          7.0.4                   Hardened WordPress CMS (PHP-FPM + nginx) on a 0...
https://artifacthub.io/packages/helm/kube-wordp...      0.1.0           1.1                     this is my wordpress package
https://artifacthub.io/packages/helm/wordpress-...      1.0.2           1.0.0                   A Helm chart for deploying Wordpress+Mariadb st...
...
```

Artifact Hub is the community-driven chart repository, but there are other repositories as well, like the Bitnami Helm repository.  
To search for charts in other repositories, you must first add a repository to the local Helm setup

```console
$ helm repo add bitnami https://charts.bitnami.com/bitnami
$ helm search repo wordpress
NAME                                    CHART VERSION   APP VERSION     DESCRIPTION
bitnami/wordpress                       30.0.9          6.9.4           WordPress is the world's most popular blogging ...
bitnami/wordpress-intel                 2.1.31          6.1.1           DEPRECATED WordPress for Intel is the most popu...
helm-helm-stable-proxy/wordpress        9.0.3           5.3.2           DEPRECATED Web publishing platform for building...
```

```console
$ helm repo list
NAME      URL
bitnami   https://charts.bitnami.com/bitnami
```

After finding the chart, the next step is to install it on the cluster with the `helm install` command, followed by the release name and the chart name.  
When this command is run, the Helm chart is downloaded from the repository and extracted, and its resources are then installed on the cluster.

Each installation of a chart is called a "Release"; each release has a release name (which is the one you specify with the `helm install` command).  
For instance, you can install the same application using the same chart multiple times by changing the release name each time you run the command

```bash
helm install release-1 bitnami/wordpress
helm install release-2 bitnami/wordpress
helm install release-3 bitnami/wordpress
```

In the case above, we have three releases of the same chart.

To list installed packages, run the `helm list` command

```console
$ helm list
NAME            NAMESPACE       REVISION        UPDATED                                         STATUS          CHART                   APP VERSION
release-1       default         1               2026-08-24 10:52:08.679212786 +0200 CEST        deployed        wordpress-30.0.9        6.9.4
release-2       default         1               2026-08-24 10:52:15.301456789 +0200 CEST        deployed        wordpress-30.0.9        6.9.4
release-3       default         1               2026-08-24 10:52:21.933218456 +0200 CEST        deployed        wordpress-30.0.9        6.9.4
```

> [!NOTE]
>
> Chart version and app version are different concepts.  
> A chart is the package; a release is a deployed instance of that chart.

to uninstall packages, run the `helm uninstall` command

```console
$ helm uninstall release-1
release "release-1" uninstalled
```

you can also download the chart without uninstalling with the `helm pull` command

```console
$ helm pull --untar bitnami/wordpress
$ ls wordpress
Chart.yaml  README.md  charts  templates  values.yaml
```

The goal is to manually edit the `values.yaml` file to change the required values, and then install it later by specifying the path to that particular directory.

```bash
helm install release-4 ./wordpress
```

Helm maintains a Release associated with each installation; then, each `helm upgrade` creates a new revision, and you can roll back to a specific revision with the `helm rollback` command.

```text
Chart + Values
      │
      ▼
   Release
      │
      ├── revision 1
      ├── revision 2
      └── revision 3
```

The following file is a Helm template rather than a static Kubernetes manifest.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ .Values.name }}
spec:
  replicas: {{ .Values.replicaCount }}
  selector:
    matchLabels:
      app: {{ .Values.name }}
  template:
    metadata:
      labels:
        app: {{ .Values.name }}
    spec:
      containers:
        - name: {{ .Chart.Name }}
          image: "nginx:{{ .Values.image.tag }}"
```

The `{{ ... }}` expressions are Helm template expressions that are evaluated when the chart is rendered or installed. Their values can come from the chart's `values.yaml` file or from additional values files or command-line options.  
`.Values` contains values provided by the chart configuration, while `.Chart` exposes metadata defined in `Chart.yaml`.  
For example, `{{ .Values.replicaCount }}` is replaced with the configured number of replicas, while `{{ .Values.image.tag }}` determines which NGINX image tag is used.  
The same template can therefore be reused across different environments without duplicating the Kubernetes manifest. Only the values that change between environments need to be provided separately.

A common way to organize a Helm-based application is to keep reusable Kubernetes templates separate from environment-specific configuration.

```text
my-chart/
├── Chart.yaml
├── values.yaml
└── templates/
    ├── nginx.deployment.yaml
    ├── nginx.service.yaml
    ├── db.deployment.yaml
    └── db.service.yaml

environments/
├── values.dev.yaml
├── values.staging.yaml
└── values.production.yaml
```

The `templates/` directory contains the Kubernetes resource templates, which are shared across environments. The `environments/` directory contains values files that provide different configuration for development, staging, and production.  
For example, the same `nginx.deployment.yaml` template can be rendered using `values.dev.yaml` for development and `values.production.yaml` for production. This allows the Kubernetes resources to remain consistent while environment-specific settings such as replica counts, image tags, or resource limits can vary.

Useful commands:

- `helm --help`: get information on the tool, the available actions, variables and commands;
- `helm env`: retrieve the Helm client environment information;
- `helm version`: get information on the Helm version;
- `--debug`: flag that can be added to other Helm commands (e.g. `helm install --debug`) to enable verbose output;
- `helm install <release> <chart> -f values.yaml`: install the application defined in the `values.yaml`;
- `helm install --dry-run <release> <chart>`: simulate the chart installation without actually performing it;
- `helm template <release> <chart>`: render the chart locally (it generates the Kubernetes manifests) without installing it (e.g. `helm template wordpress ./wordpress` or `helm template wordpress bitnami/wordpress`);
- `helm history <release>`: see the different revisions of a release;
- `helm status <release>`: check the status of the release;
- `helm get values <release>`: retrieve the values used by a given release;
- `helm upgrade <release> <chart> -f values.yaml`: modify the application with the new values provided in the `values.yaml` file;
- `helm rollback <release>`: roll back to the previous revision;
- `helm rollback <release> [REVISION]`: roll back to a specific revision;
- `helm uninstall <release>`: uninstall the application;
- `helm list -A`: list all the releases in all namespaces;
- `helm search hub <words-to-search>`: search charts in the Artifact Hub;
- `helm search repo <words-to-search>`: search charts in the locally configured repositories;
- `helm show values <chart>`: check the values.yaml of a chart before installing it;
- `helm pull <chart>`: download the chart as an archive;
- `helm pull --untar <chart>`: download the chart and extract it;
- `helm repo add <repo-name> <repo-url>`: add a new chart repository, e.g. `helm repo add bitnami https://charts.bitnami.com/bitnami`;
- `helm repo list`: list the locally configured repositories;
- `helm repo update`: update the local indexes of the repositories.

---

## Kustomize Basics

Kustomize lets us customize existing Kubernetes manifests without modifying the original YAML files.  
This is particularly useful when the same application needs different configurations for different environments: it allows to reuse our configurations and only modify what needs to be changed on a per-environment basis (avoiding to repeat the configuration manifests both for the common and different parts).

Kustomize comes built-in within the Kubectl; anyway, you may want to install Kustomize CLI separately to get the latest version.

---

### The kustomization.yaml file and the folder structure

The `kustomization.yaml` file describes:

- the resources that should be included;
- the transformations and customizations that should be applied to them.

> [!TIP]
>
> Just like any other Kubernetes resource file, you can set the `apiVersion` and `kind` properties on a `kustomization.yaml` file
>
> ```yaml
> apiVersion: kustomize.config.k8s.io/v1beta1
> kind: Kustomization
>
> resources:
> - nginx-deployment.yaml
> - nginx-service.yaml
>
> commonLabels:
>   goal: CKAD
> ```
>
> They are technically optional, Kustomize will pick up default values, but you can hardcode those values in order to prevent breaking changes in the future.

We then need to define the folder structure in order to correctly run the `kustomize build <directory>` command, which will:

1) import the resources defined in the manifest files;
2) apply the transformations defined in the kustomization file;
3) return the final configurations.

For example, the following one is a compliant structure

```yaml
kubernetes_manifests/
├── nginx-deployment.yaml
├── nginx-service.yaml
└── kustomization.yaml
```

with the following contents

```yaml
# Kubernetes resources to be managed by Kustomize
resources:
- nginx-deployment.yaml
- nginx-service.yaml

# Customizations that need to be made
commonLabels:
  goal: CKAD
```

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
spec:
  replicas: 1
  selector:
    matchLabels:
      component: nginx
  template:
    metadata:
      labels:
        component: nginx
    spec:
      containers:
      - image: nginx
        name: nginx
```

```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-loadbalancer-service
spec:
  ports:
  - port: 80
    protocol: TCP
    targetPort: 3000
  selector:
    component: nginx
  type: LoadBalancer
```

---

### Building and applying Kustomize configurations

When running `kustomize build <directory>`, Kustomize looks for a file named, by default, `kustomization.yaml`, in the specified directory.

The command `kustomize build kubernetes_manifests/`, will generate the manifests as follows

```console
$ kustomize build kubernetes_manifests
apiVersion: v1
kind: Service
metadata:
  labels:
    goal: CKAD
  name: nginx-loadbalancer-service
spec:
  ports:
  - port: 80
    protocol: TCP
    targetPort: 3000
  selector:
    goal: CKAD
    component: nginx
  type: LoadBalancer
---
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    goal: CKAD
  name: nginx-deployment
spec:
  replicas: 1
  selector:
    matchLabels:
      goal: CKAD
      component: nginx
  template:
    metadata:
      labels:
        goal: CKAD
        component: nginx
    spec:
      containers:
      - image: nginx
        name: nginx
```

Note that the Kustomize command does not apply or deploy those Kubernetes resources; it instead only returns the rendered result.  
If we want to apply those configurations to a cluster, we need to redirect this output in the `kubectl apply` command, for example with the use of the "pipe" Linux utility as follows

```bash
kustomize build kubernetes_manifests/ | kubectl apply -f -
```

or, you can also do this natively with the command

```bash
kubectl apply -k kubernetes_manifests/
```

You can also delete the declared resources with the `delete` commands

```bash
# Piping
kustomize build kubernetes_manifests/ | kubectl delete -f -
# Natively
kubectl delete -k kubernetes_manifests/
```

---

### Organizing resources into directories

In some cases, we would like to separate configuration parts in different folders in a way that makes more sense

```text
kubernetes/
├───api/
│   ├───api-deployment.yaml
│   └───api-service.yaml
└───db/
    ├───db.deployment.yaml
    └───db.service.yaml
```

Now we ne can perfrom `kubectl` commands relatively to a directory and will be performed to all the files inside the folder

```bash
kubectl apply -f kubernetes/api
kubectl apply -f kubernetes/db
```

If the structure grows (quantity of directories and subdirectory, number of resources managed), it starts to get difficult to apply what is needed to a specific environment and application.  
We can instead create a `kustomization.yaml` file in the root of our directory and list there all the resources we want to manage

```text
kubernetes_manifests/
├───kustomization.yaml
├───api/
│   ├───api-deployment.yaml
│   └───api-service.yaml
└───db/
    ├───db.deployment.yaml
    └───db.service.yaml
```

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
- api/api-deployment.yaml
- api/api-service.yaml
- db/db-deployment.yaml
- db/db-service.yaml
```

And so when applying this configuration it will only be just one command, without needing to go in each of the required subdirectories

```bash
# Piping
kustomize build kubernetes_manifests/ | kubectl apply -f -
# Natively
kubectl apply -k kubernetes_manifests/
```

If the number of directories grows excessively, the `resources` list contained in the `kustomization.yaml` file will start to get fairly long.

Although this is technically a valid solution, a way to manage this is adding separate `kustomization.yaml` files in the required subdirectories

```text
kubernetes_manifests/
├───kustomization.yaml
├───api/
│   ├───api-deployment.yaml
│   ├───api-service.yaml
│   └───kustomization.yaml
├───db/
│   ├───db-deployment.yaml
│   ├───db-service.yaml
│   └───kustomization.yaml
├───cache/
│   ├───cache-deployment.yaml
│   ├───cache-service.yaml
│   └───kustomization.yaml
└───kafka/
    ├───kafka.deployment.yaml
    ├───kafka.service.yaml
    └───kustomization.yaml
```

in which we can define the resources with a more granular criteria.  
In this way, `kubernetes_manifests/db/kustomization.yaml` will be, for example

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
- db-deployment.yaml
- db-service.yaml
```

Then, in the root `kubernetes_manifests/kustomization.yaml` we'll provide the paths of the different directories that we want to include

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
- api/
- db/
- cache/
- kafka/
```

and when we do that, Kustomize will look for a `kustomization.yaml` file in those directories and import all of those.  
This approach will allow us to keep the structure as clean as possible and to apply the changes still with one command.

---

### Transformers

Kustomize has several built-in transformers (you can even create your own).

Here some example of the default transformers:

- **commonLabels**: adds a label to all Kubernetes resources;
- **namePrefix/Suffix**: adds a common prefix or suffix to all resource names;
- **Namespace**: adds a common namespace to all resources;
- **commonAnnotations**: adds an annotation to all Kubernetes resources;
- **images**: updates the specified image key values found in paths that include `containers` and `initcontainers` sub-paths. If found, the `image` key value is customized by the values set in the `newName`, `newTag`, and `digest` fields. The `name` field should match the `image` key value in a resource.

> [!NOTE]
>
> `commonLabels` can also update selectors where appropriate.  
> This is useful for keeping labels and selectors consistent,
> but it should be used carefully because changing selectors can affect how resources identify their Pods.

Let's make some examples starting from the following definition file

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
spec:
  replicas: 1
  selector:
    matchLabels:
      component: nginx
  template:
    metadata:
      labels:
        component: nginx
    spec:
      containers:
      - image: nginx
        name: nginx
```

We already seen the `commonLabel` transformer in action.

With the `namespace` transformer, applying the following kustomization.yaml

```yaml
namespace: lab
```

will result in the following output

```diff
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
+ namespace: lab
spec:
  replicas: 1
  selector:
    matchLabels:
      component: nginx
  template:
    metadata:
      labels:
        component: nginx
    spec:
      containers:
      - image: nginx
        name: nginx
```

With the `namePrefix/Suffix` transformer instead, in this example we want to add the prefix `CKAD-` and the suffix `-dev`.  
So with the following kustomization.yaml

```yaml
namePrefix: CKAD-
nameSuffix: -dev
```

the result will be this one

```diff
apiVersion: apps/v1
kind: Deployment
metadata:
- name: nginx-deployment
+ name: CKAD-nginx-deployment-dev
spec:
  replicas: 1
  selector:
    matchLabels:
      component: nginx
  template:
    metadata:
      labels:
        component: nginx
    spec:
      containers:
      - image: nginx
        name: nginx
```

Making an example of the `commonAnnotations` transformer, with the following kustomization.yaml

```yaml
commonAnnotations:
  branch: master
```

the result will be this one

```diff
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
+ annotations:
+   branch: master
spec:
  replicas: 1
  selector:
    matchLabels:
      component: nginx
  template:
    metadata:
+     annotations:
+       branch: master
      labels:
        component: nginx
    spec:
      containers:
      - image: nginx
        name: nginx
```

If using the `image` transformer for changing the image repository from `nginx` to `haproxy`, we can use this configuration

```yaml
images:
  - name: nginx
    newName: haproxy
```

and have this result

```diff
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
spec:
  replicas: 1
  selector:
    matchLabels:
      component: nginx
  template:
    metadata:
      labels:
        component: nginx
    spec:
      containers:
-     - image: nginx
+     - image: haproxy
        name: nginx
```

If we instead only want to change the tag of the image, without changing the image repository, we can still do that with the `image` transformer with the following configuration

```yaml
images:
  - name: nginx
    newTag: 2.4
```

and obtain this result

```diff
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
spec:
  replicas: 1
  selector:
    matchLabels:
      component: nginx
  template:
    metadata:
      labels:
        component: nginx
    spec:
      containers:
-     - image: nginx
+     - image: nginx:2.4
        name: nginx
```

With the `image` transformer we can also override both the image repository and its tag, the digest and any combination; also, we can set more than one override in the `kustomization.yaml`

```yaml
images:
  - name: postgres
    newName: my-registry/my-postgres
    newTag: v1
  - name: nginx
    newTag: 1.8.0
  - name: my-demo-app
    newName: my-app
  - name: alpine
    digest: sha256:25a0d4
```

---

### Patches

Kustomize patches provide another method to modify Kubernetes configurations.

Unlike default transformers, patches provide a more "surgical" approach to target one or more specific sections in a Kubernetes resource.  
This is useful in situations like when we need to update the number of replicas in a specific `Deployment`.

To create a patch we need to provide 3 parameters:

- **operation type**: the most used are "add", "remove" or "replace";
- **target**: what resource should this patch be applied on:
  - Kind;
  - API version / group;
  - name;
  - namespace;
  - labelSelector;
  - annotationSelector;
- **value**: what is the value that will either be replaced or added with (only needed for add or replace operations).

Let's make some examples starting from the same Deployment definition as before

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
spec:
  replicas: 1
  selector:
    matchLabels:
      component: nginx
  template:
    metadata:
      labels:
        component: nginx
    spec:
      containers:
      - image: nginx
        name: nginx
```

If we want to change the `metadata.name` to `web-deployment`, we'll add the following content to the `kustomization.yaml`

```yaml
patches:
  - target:
      kind: Deployment
      name: nginx-deployment
    patch: |-
      - op: replace
        path: /metadata/name
        value: web-deployment
```

If we want to change the `spec.replicas` to `5`, we'll add the following content to the `kustomization.yaml`

```yaml
patches:
  - target:
      kind: Deployment
      name: nginx-deployment
    patch: |-
      - op: replace
        path: /spec/replicas
        value: 5
```

> [!NOTE]
>
> The `|-` part is for stating that we are applying an "inline patch"

This is the structure of a **JSON 6902** type of patch;  
There is also another way, which is the **strategic merge patch**, in which we only put the manifest without the stuff that we don't want to change.

```yaml
patches:
  - patch: |-
      apiVersion: apps/v1
      kind: Deployment
      metadata:
        name: nginx-deployment
      spec:
        replicas: 5
```

For both **JSON 6902** and **strategic merge patch**, there are two different ways to define a patch:  
The first one is inline, with which we've been working until now, where we define the patch inside the same `Kustomization.yaml`;  
the other one is by providing a separate file in which we'll provide the patch, for example

- with the **JSON 6902** using the following `Kustomization.yaml`

  ```yaml
  patches:
    - path: replica-patch.yaml
      target:
        kind: Deployment
        name: nginx-deployment
  ```

  and the following `replica-patch.yaml`

  ```yaml
  - op: replace
    path: /spec/replicas
    value: 5
  ```

- with the **strategic merge patch** using the following `Kustomization.yaml`

  ```yaml
  patches:
    - replica-patch.yaml
  ```

  and the following `replica-patch.yaml`

  ```yaml
  apiVersion: apps/v1
  kind: Deployment
  metadata:
    name: nginx-deployment
  spec:
    replicas: 5
  ```

Let's say now that we want to replace the value of `spec.template.metadata.labels.component` to `web`

- this is what we need using an inline JSON 6902 patch

  ```yaml
  patches:
    - target:
        kind: Deployment
        name: nginx-deployment
      patch: |-
        - op: replace
          path: /spec/template/metadata/labels/component
          value: web
  ```

- this is what we need using a multi-file strategic merge patch:

  ```yaml
  patches:
    - label-patch.yaml
  ```

  ```yaml
  apiVersion: apps/v1
  kind: Deployment
  metadata:
    name: nginx-deployment
  spec:
    template:
      metadata:
        labels:
          component: web
  ```

In this example we'll add a new key/value `goal: CKAD` in `spec.template.metadata.labels`

- this is what we need using an inline JSON 6902 patch

  ```yaml
  patches:
    - target:
        kind: Deployment
        name: nginx-deployment
      patch: |-
        - op: add
          path: /spec/template/metadata/labels/goal
          value: CKAD
  ```

- this is what we need using a multi-file strategic merge patch:

  ```yaml
  patches:
    - label-patch.yaml
  ```

  ```yaml
  apiVersion: apps/v1
  kind: Deployment
  metadata:
    name: nginx-deployment
  spec:
    template:
      metadata:
        labels:
          goal: CKAD
  ```

In this example we'll remove the `spec.template.metadata.labels` `goal: CKAD` set in the example above

- this is what we need using an inline JSON 6902 patch

  ```yaml
  patches:
    - target:
        kind: Deployment
        name: nginx-deployment
      patch: |-
        - op: remove
          path: /spec/template/metadata/labels/goal
  ```

- this is what we need using a multi-file strategic merge patch:

  ```yaml
  patches:
    - label-patch.yaml
  ```

  ```yaml
  apiVersion: apps/v1
  kind: Deployment
  metadata:
    name: nginx-deployment
  spec:
    template:
      metadata:
        labels:
          goal: null
  ```

> [!TIP]
>
> To remove a field using a strategic merge patch, it can be set to `null`.

Until now we have seen how to add, replace or remove keys to a dictionary; let's now see how to perform those operations on a list.

Starting from the manifest given above

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
spec:
  replicas: 1
  selector:
    matchLabels:
      component: nginx
  template:
    metadata:
      labels:
        component: nginx
    spec:
      containers:
      - image: web
        name: nginx
```

If we want to change the name and the image of the container

- this is what we need using an inline JSON 6902 patch

  ```yaml
  patches:
    - target:
        kind: Deployment
        name: nginx-deployment
      patch: |-
        - op: replace
          path: /spec/template/spec/containers/0
          value:
            name: haproxy
            image: haproxy
  ```

- this is what we need using a multi-file strategic merge patch, using the following `Kustomization.yaml`

  ```yaml
  patches:
    - image-patch.yaml
  ```

  and the following `image-patch.yaml`:

  ```yaml
  apiVersion: apps/v1
  kind: Deployment
  metadata:
    name: nginx-deployment
  spec:
    template:
      spec:
        containers:
          - name: web
            image: haproxy
  ```

> [!NOTE]
>
> The number at the end of `path: /spec/template/spec/containers/0` specifies the index of the object to manipulate.

If we want to add a second container on that list

- this is what we need using an inline JSON 6902 patch

  ```yaml
  patches:
    - target:
        kind: Deployment
        name: nginx-deployment
      patch: |-
        - op: add
          path: /spec/template/spec/containers/-
          value:
            name: web2
            image: haproxy
  ```

  > [!NOTE]
  >
  > The `-` at the end of `path: /spec/template/spec/containers/-` means that we want to add the container item at the end of the list.  
  > If we wanted to add it at the beginning of the list, we should use the `0` index.

- this is what we need using a multi-file strategic merge patch, using the following `Kustomization.yaml`

  ```yaml
  patches:
    - image-patch.yaml
  ```

  and the following `image-patch.yaml`:

  ```yaml
  apiVersion: apps/v1
  kind: Deployment
  metadata:
    name: nginx-deployment
  spec:
    template:
      spec:
        containers:
          - name: web2
            image: haproxy
  ```

  > [!NOTE]
  >
  > For `containers`, the `name` field is used as the merge key.  
  > If the patch contains a container whose `name` matches an existing container, that container is merged with the patch.  
  > If no matching container exists, a new container is added.  
  >
  > Since in this case no existing container has the name `web2`, the container is added rather than merged with an existing one.

Now, starting from the following manifest

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api-deployment
spec:
  replicas: 1
  selector:
    matchLabels:
      component: nginx
  template:
    metadata:
      labels:
        component: nginx
    spec:
      containers:
      - image: nginx
        name: nginx
      - image: mongo
        name: database
```

we'll see how to delete the second container from that list.

- this is what we need using an inline JSON 6902 patch

  ```yaml
  patches:
    - target:
        kind: Deployment
        name: api-deployment
      patch: |-
        - op: remove
          path: /spec/template/spec/containers/1
  ```

- this is what we need using a multi-file strategic merge patch, using the following `Kustomization.yaml`

  ```yaml
  patches:
    - remove-container-patch.yaml
  ```

  and the following `remove-container-patch.yaml`:

  ```yaml
  apiVersion: apps/v1
  kind: Deployment
  metadata:
    name: api-deployment
  spec:
    template:
      spec:
        containers:
          - $patch: delete
            name: database
  ```

  > [!NOTE]
  >
  > In this case, considering that we cannot list something that we want to delete within the strategic merge, we have to use the delete directive `$patch: delete`.

---

### Base and overlays

Kustomize commonly uses a base and overlays structure to manage configurations that are shared across multiple environments:

- **base**: the base configuration, which represents the common part across the different environments, along with the default values;
- **overlays**: environment-specific configurations that customize the base according to the requirements of each environment.

The combination of a base and an overlay generates the final manifests.

> [!NOTE]
>
> `base` and `overlays` are not fundamental mechanisms of Kustomize: they are instead a common pattern for organizing Kustomization files.

When using this pattern, the folder structure can follow a structure similar to the following:

- a `base` folder, containing the common configuration shared across the different environments;
- an `overlays` folder, with sub-folders representing the different environments and containing the customizations and resources specific to each environment.

```text
kubernetes/
├───base/
│   ├───kustomization.yaml
│   ├───nginx-deploy.yaml
│   ├───service.yaml
│   └───redis-deploy.yaml
└───overlays/
    ├───dev/
    │   ├───kustomization.yaml
    │   └───config-map.yaml
    ├───staging/
    │   ├───kustomization.yaml
    │   └───config-map.yaml
    └───prod/
        ├───kustomization.yaml
        └───config-map.yaml
```

We customize each environment by applying one or more patches on top of the base.

For example, in order to differentiate the nginx `Deployment` you can do as follows:

- `base/kustomization.yaml`

  ```yaml
  resources:
    - nginx-deploy.yaml
    - service.yaml
    - redis-depl.yaml
  ```

- `base/nginx-deploy.yaml`

  ```yaml
  apiVersion: apps/v1
  kind: Deployment
  metadata:
    name: nginx-deployment
  spec:
    replicas: 1
  ```

- `overlays/dev/kustomization.yaml`

  ```yaml
  resources:
    - ../../base

  patches:
    - target:
        kind: Deployment
        name: nginx-deployment
      patch: |-
        - op: replace
          path: /spec/replicas
          value: 2
  ```

- `overlays/prod/kustomization.yaml`

  ```yaml
  resources:
    - ../../base

  patches:
    - target:
        kind: Deployment
        name: nginx-deployment
      patch: |-
        - op: replace
          path: /spec/replicas
          value: 3
  ```

so that the deployment in the dev environment will have two replicas while in production will have three.

You can also define resource that belong to a specific environment, so for example adding a Grafana instance only on the production enviromnet by

- **adding** a `overlays/prod/grafana-deploy.yaml` Grafana Deployment manifest

- having the following `overlays/prod/kustomization.yaml`

  ```yaml
  resources:
    - ../../base
    - grafana-deploy.yaml

  patches:
    - target:
        kind: Deployment
        name: nginx-deployment
      patch: |-
        - op: replace
          path: /spec/replicas
          value: 2
  ```

The `base` and `overlays` directory names are not mandatory. The directory structure can be organized according to the needs of the project.

```text
kubernetes/
├── base/
│   ├── kustomization.yaml
│   ├── db/
│   │   ├── db-depl.yaml
│   │   ├── db-service.yaml
│   │   └── kustomization.yaml
│   └── api/
│       ├── api-depl.yaml
│       ├── api-service.yaml
│       └── kustomization.yaml
└── overlays/
    ├── dev/
    │   ├── kustomization.yaml
    │   ├── db/
    │   │   ├── db-patch.yaml
    │   │   └── kustomization.yaml
    │   └── api/
    │       ├── api-patch.yaml
    │       └── kustomization.yaml
    └── prod/
        ├── kustomization.yaml
        ├── db/
        │   ├── db-patch.yaml
        │   └── kustomization.yaml
        ├── api/
        │   ├── api-patch.yaml
        │   └── kustomization.yaml
        └── grafana/
            ├── grafana-deploy.yaml
            └── kustomization.yaml
```

---

### Components

Components allow us to define reusable pieces of configuration logic that can be included in multiple overlays.  
They are useful when an application has optional features that need to be enabled only in a subset of environments.  
Unlike overlays, Components are not tied to a specific environment.

For example, we could define a component for monitoring, another one for external database support, and another one for LDAP authentication.  
Each component can contain its own resources and patches.

A component is defined using a `kustomization.yaml` file with `kind: Component`:

```yaml
apiVersion: kustomize.config.k8s.io/v1alpha1
kind: Component

resources:
  - monitoring-config.yaml

patches:
  - monitoring-patch.yaml
```

The component can then be included by an overlay using the `components` field

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
  - ../../base

components:
  - ../../components/monitoring
```

In this way, the same optional functionality can be reused across multiple overlays without duplicating its configuration.

In order to understand better the components, let's make another example considering an application that is available in three different configurations:

- **Community Edition**: the development/community version of the application;
- **Premium**: the premium cloud version of the application;
- **Self Hosted**: the version that is deployed and managed by the customer.

All three configurations share a common base, but each one is represented by a different overlay.

Some optional features, however, need to be enabled only for certain configurations:

- an **external database** is required by both cloud versions (Community Edition and Premium);
- **caching** is available for Premium and Self Hosted, but not for Community Edition.

Instead of duplicating the configuration for these features in each overlay, we can define them as reusable Components.

![Kustomize Components  example](./images/06-kustomize-components-diagram.png "Kustomize Components example")

For example, we can organize the configuration as follows:

```text
k8s/
├── base/
│   ├── kustomization.yaml
│   └── application-deployment.yaml
│
├── overlays/
│   ├── community/
│   │   └── kustomization.yaml
│   ├── premium/
│   │   └── kustomization.yaml
│   └── self-hosted/
│       └── kustomization.yaml
│
└── components/
    ├── external-db/
    │   ├── kustomization.yaml
    │   ├── deployment-patch.yaml
    │   └── postgres-deploy.yaml
    │
    └── caching/
        ├── kustomization.yaml
        ├── deployment-patch.yaml
        └── redis-deploy.yaml
```

The `external-db` Component could contain the configuration required to connect the application to an external database:

```yaml
apiVersion: kustomize.config.k8s.io/v1alpha1
kind: Component

resources:
  - postgres-deploy.yaml

secretGenerator:
  - name: postgres-cred
    literals:
      - password: postgres123

patches:
  - deployment-patch.yaml
```

> [!NOTE]
>
> Since we have a database, we defined also a secret for the password of the database, that we are gonna import using the following `deployment-patch.yaml`
>
> ```yaml
> apiVersion: apps/v1
> kind: Deployment
> metadata:
>   name: api-deployment
> spec:
>   template:
>     spec:
>       containers:
>         - name: api
>           env:
>             - name: DB_PASSWORD
>               valueFrom:
>                 secretKeyRef:
>                   name: postgres-cred
>                   key: password
> ```
>
> This is only an example. In a real-world environment, passwords and other sensitive values should not be stored directly in the Kustomization files.

The `caching` Component could instead contain the resources and configuration required to enable caching:

```yaml
apiVersion: kustomize.config.k8s.io/v1alpha1
kind: Component

resources:
  - redis-deploy.yaml

patches:
  - deployment-patch.yaml
```

The Components can then be included by the appropriate overlays:

- the Community Edition could use only the external database:

  ```yaml
  apiVersion: kustomize.config.k8s.io/v1beta1
  kind: Kustomization

  resources:
    - ../../base

  components:
    - ../../components/external-db
  ```

- the Premium version could use both Components:

  ```yaml
  apiVersion: kustomize.config.k8s.io/v1beta1
  kind: Kustomization

  resources:
    - ../../base

  components:
    - ../../components/external-db
    - ../../components/caching
  ```

- the Self Hosted version could use only the caching Component:

  ```yaml
  apiVersion: kustomize.config.k8s.io/v1beta1
  kind: Kustomization

  resources:
    - ../../base

  components:
    - ../../components/caching
  ```

This allows the same configuration logic to be reused across multiple overlays without duplicating it.

As the application evolves, the base and the individual overlays can continue to change independently, while the reusable functionality remains encapsulated in the Components. If the implementation of a feature needs to change, we can update the corresponding Component rather than duplicating the same changes across multiple overlays.  
This makes the configuration easier to maintain and scale as the number of environments and optional features grows.

Here is a comparison about bases, overlays and components.

| Aspect              | Base                 | Overlay                            | Component                         |
| ------------------- | -------------------- | ---------------------------------- | --------------------------------- |
| Purpose             | Shared configuration | Environment-specific customization | Reusable optional functionality   |
| Reusable            | Yes                  | Usually not                        | **Yes, across multiple overlays** |
| Contains resources  | Yes                  | Yes                                | Yes                               |
| Can contain patches | Yes                  | Yes                                | Yes                               |
| Included using      | `resources`          | `resources`                        | `components`                      |

---

### Kustomize vs Helm

Helm and Kustomize partly address the same issue: providing a way to modify Kubernetes applications on a per-environment basis.  
In order to understand which fits best for our needs between Helm or Kustomize, we need to understand some differences.

Kustomize works directly with Kubernetes YAML manifests, so the resulting resources remain standard Kubernetes manifests.  
Helm, instead, makes use of Go templates syntax in order to assign variables to properties; therefore, Helm templates are not valid YAML and can easily become hard to read / interpretate.

The projects structure between the two tools is slightly different.

Helm is more than just a tool to customize configurations on a per-environment basis; it is also a complete package manager for the application.

Kustomize does not require learning complex templating systems, so it's easier and simple,  
whereas Helm is a little bit more complex but carries extra features as well (like conditionals, loops, functions and hooks).

|                       | Kustomize                      | Helm               |
| --------------------- | ------------------------------ | ------------------ |
| Input                 | Kubernetes YAML                | Templates          |
| Modifies              | Overlay / patch / transformers | Values + templates |
| Template language     | No                             | Go templates       |
| Packaging             | No                             | Yes                |
| Package manager       | No                             | Yes                |
| `values.yaml`         | No                             | Yes                |
| Releases              | No                             | Yes                |
| Rollback capabilities | No                             | Yes                |
| Difficulty            | Lower                          | Higher             |

---

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
