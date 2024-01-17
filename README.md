# Despliegue de Spark con Kubernetes

## Clonar este repositorio

Clonar el repositorio. Nos aseguraremos de que nuestro directorio de trabajo es ~/spark-minikube. 

## Puesta en marcha de Minikube

Si no está ya hecho, instalar [Minikube](https://kubernetes.io/docs/setup/minikube/). Iniciamos Minikube con:

```sh
minikube start
eval $(minikube docker-env)
```

## Puesta en marcha del clúster Spark

Construir la imagen Docker image:

```sh
docker build -t spark-hadoop:3.2.0 -f ./docker/Dockerfile ./docker
```

Alternativa más rápida a "docker build": descarga la imagen la lista de Docker Hub y reetiquétala:
```sh
docker pull acpmialj/ipmd:spark_hadoop_3.2.0
docker tag acpmialj/ipmd:spark_hadoop_3.2.0 spark-hadoop:3.2.0
```


Creamos deployments y services:

```sh
kubectl create -f ./kubernetes/spark-master-deployment.yaml
kubectl create -f ./kubernetes/spark-master-service.yaml
kubectl create -f ./kubernetes/spark-worker-deployment.yaml
minikube addons enable ingress
# Es posible que el siguiente comando haya que ejecutarlo un par de veces, tarda
kubectl apply -f ./kubernetes/minikube-ingress.yaml
```

Vemos nuestros contenedores. Nos fijamos en el nombre y la dirección IP del que empieza por spark-master:
```sh
kubectl get pods -o wide

NAME                            READY   STATUS    RESTARTS   AGE     IP           NODE       NOMINATED NODE   READINESS GATES
spark-master-dbc47bc9-t6v84     1/1     Running   0          7m35s   172.17.0.6   minikube   <none>           <none>
spark-worker-795dc47587-5ch8f   1/1     Running   0          7m24s   172.17.0.9   minikube   <none>           <none>
spark-worker-795dc47587-fvcf6   1/1     Running   0          7m24s   172.17.0.7   minikube   <none>           <none>
```

Si todo va bien, podemos hacer una modificación en /etc/hosts para poder conectarnos al Web UI del Spark master:
```sh
echo "$(minikube ip) spark-kubernetes" | sudo tee -a /etc/hosts
```
Y luego conectarnos vía web a: http://spark-kubernetes/. 

Lanzamos pyspark en el master. Tenemos que usar el nombre y la dirección IP correspondiente:

```sh
kubectl exec spark-master-dbc47bc9-t6v84 -it -- \
    pyspark --conf spark.driver.bindAddress=172.17.0.6 --conf spark.driver.host=172.17.0.6
```

Una vez veamos el prompt, (si no aparece, pulsar Return) podremos escribir código Python.

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

Podemos hacer operaciones de limpieza

```sh
$ kubectl get deployment
NAME           READY   UP-TO-DATE   AVAILABLE   AGE
spark-master   1/1     1            1           65m
spark-worker   2/2     2            2           65m

$ kubectl get service
NAME           TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)             AGE
kubernetes      ClusterIP   10.96.0.1       <none>        443/TCP             104d
spark-master    ClusterIP   10.103.199.43   <none>        8080/TCP,7077/TCP   13m

$ kubectl get ingress
NAME               CLASS   HOSTS              ADDRESS        PORTS   AGE
minikube-ingress   nginx   spark-kubernetes   192.168.49.2   80      12m

$ kubectl delete deployment spark-master spark-worker
deployment.apps "spark-master" deleted
deployment.apps "spark-worker" deleted

$ kubectl delete service spark-master
service "spark-master" deleted

$ kubectl delete ingress minikube-ingress
ingress.networking.k8s.io "minikube-ingress" deleted

$ minikube stop # or minikube delete
✋  Stopping node "minikube"  ...
🛑  Powering off "minikube" via SSH ...
🛑  1 node stopped.
```

## Fuente

Ver [post](https://testdriven.io/deploying-spark-on-kubernetes).
