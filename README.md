# Deploying Spark on Kubernetes

This post details how to deploy Spark on a Kubernetes cluster.

*Dependencies:*

- Docker v18.06.1-ce
- Minikube v0.29.0
- Spark v2.2.1
- Hadoop 2.7.3

## Minikube

[Minikube](https://kubernetes.io/docs/setup/minikube/) is a tool used to run a single-node Kubernetes cluster locally.

Follow the official [Install Minikube](https://kubernetes.io/docs/tasks/tools/install-minikube/) guide to install it along with a [Hypervisor](https://kubernetes.io/docs/tasks/tools/install-minikube/#install-a-hypervisor) (like [VirtualBox](https://www.virtualbox.org/wiki/Downloads) or [HyperKit](https://github.com/moby/hyperkit), to manage virtual machines, and [Kubectl](https://kubernetes.io/docs/tasks/tools/install-kubectl/), to deploy and manage apps on Kubernetes.

By default, the Mikikube VM is configured to use 1GB of memory and 2 CPU cores. This is [not sufficient](https://spark.apache.org/docs/2.3.1/hardware-provisioning.html) for Spark jobs, so be sure to increase the memory in your Docker [client](https://docs.docker.com/docker-for-mac/#advanced) (for HyperKit) or directly in VirtualBox. Then, when you start Mikikube, pass the memory and CPU options to it:

```sh
$ minikube start --vm-driver=hyperkit --memory 8192 --cpus 4

or

$ minikube start  --memory 8192 --cpus 4
```

## Docker

Next, let's build a custom Docker image for Spark [2.2.1](https://spark.apache.org/releases/spark-release-2-2-2.html), designed for Spark [Standalone mode](https://spark.apache.org/docs/latest/spark-standalone.html).

*Dockerfile*:

```
# base image
FROM java:openjdk-8-jdk

# define spark and hadoop versions
ENV HADOOP_VERSION 2.7.3
ENV SPARK_VERSION 2.2.1

# download and install hadoop
RUN mkdir -p /opt && \
    cd /opt && \
    curl http://archive.apache.org/dist/hadoop/common/hadoop-${HADOOP_VERSION}/hadoop-${HADOOP_VERSION}.tar.gz | \
        tar -zx hadoop-${HADOOP_VERSION}/lib/native && \
    ln -s hadoop-${HADOOP_VERSION} hadoop && \
    echo Hadoop ${HADOOP_VERSION} native libraries installed in /opt/hadoop/lib/native

# download and install spark
RUN mkdir -p /opt && \
    cd /opt && \
    curl http://archive.apache.org/dist/spark/spark-${SPARK_VERSION}/spark-${SPARK_VERSION}-bin-hadoop2.7.tgz | \
        tar -zx && \
    ln -s spark-${SPARK_VERSION}-bin-hadoop2.7 spark && \
    echo Spark ${SPARK_VERSION} installed in /opt

# add scripts and update spark default config
ADD common.sh spark-master spark-worker /
ADD spark-defaults.conf /opt/spark/conf/spark-defaults.conf
ENV PATH $PATH:/opt/spark/bin
```

You can find the above *Dockerfile* along with the Spark config file and scripts in the [spark-kubernetes](foo) repo on GitHub.

Build the image:

```sh
$ eval $(minikube docker-env)
$ docker build -t spark-hadoop:2.2.1 .
```

> If you don't want to spend the time building the image locally, feel free to use my pre-built Spark image from [Docker Hub](https://hub.docker.com/) - `mjhea0/spark-hadoop:2.2.1`.

View:

```sh
$ docker image ls spark-hadoop

REPOSITORY          TAG                 IMAGE ID            CREATED             SIZE
spark-hadoop        2.2.1               3ebc80d468bb        3 minutes ago       875MB
```

## Spark Master

*spark-master-deployment.yaml*:

```yaml
kind: Deployment
apiVersion: extensions/v1beta1
metadata:
  name: spark-master
spec:
  replicas: 1
  selector:
    matchLabels:
      component: spark-master
  template:
    metadata:
      labels:
        component: spark-master
    spec:
      containers:
        - name: spark-master
          image: spark-hadoop:2.2.1
          command: ["/spark-master"]
          ports:
            - containerPort: 7077
            - containerPort: 8080
          resources:
            requests:
              cpu: 100m
```

*spark-master-service.yaml*:

```yaml
kind: Service
apiVersion: v1
metadata:
  name: spark-master
spec:
  ports:
    - name: webui
      port: 8080
      targetPort: 8080
    - name: spark
      port: 7077
      targetPort: 7077
  selector:
    component: spark-master
```

Create the Spark master Deployment and start the Services:

```sh
$ kubectl create -f ./kubernetes/spark-master-deployment.yaml
$ kubectl create -f ./kubernetes/spark-master-service.yaml
```

Verify:

```sh
$ kubectl get deployments

NAME                      DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
spark-master-deployment   1         1         1            1           11s


$ kubectl get pods

NAME                            READY     STATUS    RESTARTS   AGE
spark-master-698c46ff7d-vxv7r   1/1       Running   0          41s
```

## Spark Workers

*spark-worker-deployment.yaml*:

```yaml
kind: Deployment
apiVersion: extensions/v1beta1
metadata:
  name: spark-worker
spec:
  replicas: 2
  selector:
    matchLabels:
      component: spark-worker
  template:
    metadata:
      labels:
        component: spark-worker
    spec:
      containers:
        - name: spark-worker
          image: spark-hadoop:2.2.1
          command: ["/spark-worker"]
          ports:
            - containerPort: 8081
          resources:
            requests:
              cpu: 100m
```

Create the Spark worker Deployment:

```sh
$ kubectl create -f ./kubernetes/spark-worker-deployment.yaml
```

Verify:

```sh
$ kubectl get deployments
NAME           DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
spark-master   1         1         1            1           1m
spark-worker   2         2         2            2           3s


$ kubectl get pods

NAME                            READY     STATUS    RESTARTS   AGE
spark-master-698c46ff7d-vxv7r   1/1       Running   0          1m
spark-worker-c49766f54-r5p9t    1/1       Running   0          21s
spark-worker-c49766f54-rh4bc    1/1       Running   0          21s
```

## Ingress

Did you notice that we exposed the Spark web UI on port 8080? In order to access it outside the cluster, let's configure an [Ingress](https://kubernetes.io/docs/concepts/services-networking/ingress/) object.

*minikube-ingress.yaml*:

```yaml
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: minikube-ingress
  annotations:
spec:
  rules:
  - host: spark-kubernetes
    http:
      paths:
      - path: /
        backend:
          serviceName: spark-master
          servicePort: 8080
```

Enable the Ingress [addon](https://github.com/kubernetes/minikube/tree/master/deploy/addons/ingress):

```sh
$ minikube addons enable ingress
```

Create the Ingress object:

```sh
$ kubectl apply -f ./kubernetes/minikube-ingress.yaml
```

Next, you need to update your */etc/hosts* file to route requests from the host we defined, `spark-kubernetes`, to the Minikube instance.

Add an entry to /etc/hosts:

```sh
$ echo "$(minikube ip) spark-kubernetes" | sudo tee -a /etc/hosts
```

Test it out in the browser at [http://spark-kubernetes/](http://spark-kubernetes/):

TODO: add image

## Test

To test, run the PySpark shell from the the master container:

```sh
$ kubectl exec spark-master-698c46ff7d-r4tq5 -it pyspark
```

Then run the following code after the PySpark prompt appears:

```python
words = 'the quick brown fox jumps over the\
        lazy dog the quick brown fox jumps over the lazy dog'
sc = SparkContext()
seq = words.split()
data = sc.parallelize(seq)
counts = data.map(lambda word: (word, 1)).reduceByKey(lambda a, b: a + b).collect()
dict(counts)
sc.stop()
```

You should see:

```sh
{'brown': 2, 'lazy': 2, 'over': 2, 'fox': 2, 'dog': 2, 'quick': 2, 'the': 4, 'jumps': 2}
```

TODO: add video
