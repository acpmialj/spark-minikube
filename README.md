# Despliegue de Spark con Kubernetes

## Fuente

Ver [post](https://testdriven.io/deploying-spark-on-kubernetes).

## Pasos

### Puesta en marcha de Minikube

Instalar y ejecutar [Minikube](https://kubernetes.io/docs/setup/minikube/):

1. Install a  [Hypervisor](https://kubernetes.io/docs/tasks/tools/install-minikube/#install-a-hypervisor) (like [VirtualBox](https://www.virtualbox.org/wiki/Downloads) or [HyperKit](https://github.com/moby/hyperkit)) to manage virtual machines
1. Install and Set Up [kubectl](https://kubernetes.io/docs/tasks/tools/install-kubectl/) to deploy and manage apps on Kubernetes
1. Install [Minikube](https://github.com/kubernetes/minikube/releases)

Arrancar cluster:

```sh
$ minikube start
```

Construir la imagen Docker image:

```sh
$ eval $(minikube docker-env)
$ docker build -t spark-hadoop:3.2.0 -f ./docker/Dockerfile ./docker
```

Creamos deployments y services:

```sh
$ kubectl create -f ./kubernetes/spark-master-deployment.yaml
$ kubectl create -f ./kubernetes/spark-master-service.yaml
$ kubectl create -f ./kubernetes/spark-worker-deployment.yaml
$ minikube addons enable ingress
# Es posible que el siguiente comando haya que ejecutarlo un par de veces, tarda
$ kubectl apply -f ./kubernetes/minikube-ingress.yaml
```

Vemos nuestros contenedores. Nos fijamos en el nombre y la dirección IP del que empieza por spark-master:
```sh
$ kubectl get pods -o wide

NAME                            READY   STATUS    RESTARTS   AGE     IP           NODE       NOMINATED NODE   READINESS GATES
spark-master-dbc47bc9-t6v84     1/1     Running   0          7m35s   172.17.0.6   minikube   <none>           <none>
spark-worker-795dc47587-5ch8f   1/1     Running   0          7m24s   172.17.0.9   minikube   <none>           <none>
spark-worker-795dc47587-fvcf6   1/1     Running   0          7m24s   172.17.0.7   minikube   <none>           <none>
```

Lanzamos pyspark en el master. Tenemos que usar el nombre y la dirección IP correspondiente:

```sh
$ kubectl exec spark-master-dbc47bc9-t6v84 -it -- \
    pyspark --conf spark.driver.bindAddress=172.17.0.6 --conf spark.driver.host=172.17.0.6
```

Una vez veamos el prompt, podremos escribir código Python.

```sh
Welcome to
      ____              __
     / __/__  ___ _____/ /__
    _\ \/ _ \/ _ `/ __/  '_/
   /__ / .__/\_,_/_/ /_/\_\   version 3.2.0
      /_/

Using Python version 3.9.2 (default, Feb 28 2021 17:03:44)
Spark context Web UI available at http://172.17.0.3:4040
Spark context available as 'sc' (master = spark://spark-master:7077, app id = app-20221118101454-0000).
SparkSession available as 'spark'.
>>>
>>> words = 'the quick brown fox jumps over the\
...         lazy dog the quick brown fox jumps over the lazy dog'
>>> sc = SparkContext.getOrCreate()
>>> seq = words.split()
>>> data = sc.parallelize(seq)
>>> counts = data.map(lambda word: (word, 1)).reduceByKey(lambda a, b: a + b).collect()
>>> dict(counts)
{'quick': 2, 'the': 4, 'brown': 2, 'fox': 2, 'jumps': 2, 'over': 2, 'lazy': 2, 'dog': 2}
>>> sc.stop()
>>> exit()
```
¡Listo!
