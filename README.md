# Despliegue de Spark con Kubernetes
Instrucciones para Minikube. Si se usa Docker Desktop cambian algunas cosas. Ver al final de este documento. 

## Clonar este repositorio

Clonar el repositorio. Nos aseguraremos de que nuestro directorio de trabajo es ~/spark-minikube. 

## Puesta en marcha de Minikube

Si no está ya hecho, instalar [Minikube](https://kubernetes.io/docs/setup/minikube/). Iniciamos Minikube con:

```sh
minikube start
eval $(minikube docker-env)
```

## Puesta en marcha del clúster Spark

Necesitamos una imagen Docker con Spark. Podemos descargar una ya pre-configurada:

```sh
docker pull acpmialj/ipmd:spark_hadoop_3.2.0
docker tag acpmialj/ipmd:spark_hadoop_3.2.0 spark-hadoop:3.2.0
```

O bien podemos crearla con Docker build:

```sh
docker build -t spark-hadoop:3.2.0 -f ./docker/Dockerfile ./docker
```
La primera alternativa es más rápida. 

Creamos deployments y services:

```sh
kubectl create -f ./kubernetes/spark-master-deployment.yaml
kubectl create -f ./kubernetes/spark-master-service.yaml
kubectl create -f ./kubernetes/spark-worker-deployment.yaml
minikube addons enable ingress
# Es posible que el siguiente comando haya que ejecutarlo un par de veces, tarda
kubectl apply -f ./kubernetes/minikube-ingress.yaml
```

Vemos nuestros pods. Nos fijamos en el nombre y, sobre todo, en la dirección IP del que empieza por spark-master:
```sh
kubectl get pods -o wide

NAME                            READY   STATUS    RESTARTS   AGE   IP           NODE       NOMINATED NODE   READINESS GATES
spark-master-d89f64bf6-m5xwx    1/1     Running   0          97s   10.244.0.4   minikube   <none>           <none>
spark-worker-5f8bf4bbd4-6vr92   1/1     Running   0          96s   10.244.0.6   minikube   <none>           <none>
spark-worker-5f8bf4bbd4-pw6n7   1/1     Running   0          96s   10.244.0.5   minikube   <none>           <none>
```

Si todo va bien, podemos hacer una modificación en /etc/hosts para poder conectarnos al Web UI del Spark master:
```sh
echo "$(minikube ip) spark-kubernetes" | sudo tee -a /etc/hosts
```
Y luego conectarnos vía web a: http://spark-kubernetes/. 

Lanzamos pyspark en el master. Tenemos que usar la dirección IP correspondiente al pod spark-master (10.244.0.4):

```sh
kubectl exec svc/spark-master -it -- pyspark --conf spark.driver.host=10.244.0.4 
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
Spark context Web UI available at http://10.244.0.4:4040
Spark context available as 'sc' (master = spark://spark-master:7077, app id = app-20241105115225-0000).
SparkSession available as 'spark'.
>>>
>>> words = 'the quick brown fox jumps over the lazy dog the quick brown fox jumps over the lazy dog'
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
```

Todo lo anterior se puede hacer con "source ./delete.sh". 

Si lo consideramos oportuno, podemos parar o borrar el clúster Minikube:
```
$ minikube stop # or minikube delete
✋  Stopping node "minikube"  ...
🛑  Powering off "minikube" via SSH ...
🛑  1 node stopped.
```

## Instrucciones para Docker Desktop

Habilitamos el clúster Kubernetes de Docker Desktop desde su GUI. Cuando esté en marcha, abrimos un terminal Ubuntu. En el mismo, damos estos pasos

1. Habilitar el controlador de Ingress basado en NGINX. Esperar a que todo esté en marcha
```
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.10.1/deploy/static/provider/cloud/deploy.yaml
kubectl wait --namespace ingress-nginx --for=condition=ready pod --selector=app.kubernetes.io/component=controller --timeout=120s
```
2. Clonar el repositorio
```
git clone https://github.com/acpmialj/spark-minikube.git 
cd spark-minikube
```
3. Descargar la imagen
```
docker pull acpmialj/ipmd:spark_hadoop_3.2.0
docker tag acpmialj/ipmd:spark_hadoop_3.2.0 spark-hadoop:3.2.0
```
4. Desplegar el clúster: 
```
source create.sh
```
5. Ver que todo está en marcha
```
kubectl get all
kubectl get ingress
```
6. Acceder al webUI de Spark: http://spark-kubernetes/. Para que esto funcione, hemos tenido que añadir la línea "127.0.0.1 spark-kubernetes" al fichero "C:\Windows\System32\drivers\etc\hosts". Para ello abrimos primero el editor de textos notepad como administrador. 
7. Recogemos con "kubectl get pods -o wide" la dirección IP interna del spark-master (pod "spark-master-6bc899886b-nqvbb" o similar). Sea esta dirección "10.1.0.144". Ejecutamos
```
kubectl exec svc/spark-master -it -- pyspark --conf spark.driver.host=10.1.0.144
```
8. Una vez en PySpark, ejecutamos algunos comandos. Salimos con exit()
9. Limpiamos 
```
source delete.sh
```
9. Podemos eliminar el controlador ingress:
```
kubectl delete -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.10.1/deploy/static/provider/cloud/deploy.yaml
```
## Fuente

Ver [post](https://testdriven.io/deploying-spark-on-kubernetes).
