# monitor-wildfly-example
## steps
### 1. build wildfly-embedded application
1. git clone example project`todo-backend` [todo-backend](https://github.com/wildfly/quickstart/tree/main/todo-backend#run-the-backend-locally-as-a-bootable-jar)
2. in todo-backend folder  cmd : `$ mvn clean package -P bootable-jar-openshift`
3. run  Local PostgreSQL 
   ```python
     docker run --name todo-backend-db \
	  -e POSTGRES_USER=todos \
	  -e POSTGRES_PASSWORD=mysecretpassword \
	  -p 5432:5432 \
	  postgres
   ```  
4. find your virtual-machine ip ex : `docker-machine ls`
	```python
	$ docker-machine ls
		NAME          ACTIVE   DRIVER       STATE     URL                         SWARM   DOCKER      ERRORS
		wildfly-lab   -        virtualbox   Running   tcp://192.168.99.101:2376           v19.03.12
	```
5. run bootable jar app
```python
POSTGRESQL_DATABASE=todos \
  POSTGRESQL_SERVICE_HOST=<docker-machine ip:192.168.99.101> \
  POSTGRESQL_SERVICE_PORT=5432 \
  POSTGRESQL_USER=todos \
  POSTGRESQL_PASSWORD=mysecretpassword \
  POSTGRESQL_DATASOURCE=ToDos \
  java -jar target/todo-backend-bootable.jar
```
6. test app : `curl http://localhost:8080`
7. test app : `curl -X POST -H "Content-Type: application/json"  -d '{"title": "This is my first todo item!"}' http://localhost:8080/`
### 2. create Dockfile to containerize your app
1. create dockerfile
	```dockerfile
	# BUILD STAGE
	# from redhat website: https://catalog.redhat.com/software/containers/ubi8/openjdk-11/5dd6a4b45a13461646f677f4?container-tabs=gti&gti-tabs=unauthenticated
	FROM registry.access.redhat.com/ubi8/openjdk-11 
	WORKDIR /app
	
	COPY ./target/todo-backend-bootable.jar /app/todo-backend-bootable.jar
	
	# 8080 for services 、9990 for web console  
	EXPOSE 8080/tcp 9990/tcp 
	
	ENV TZ=Asia/Taipei
	ENV JAVA_OPTS=""
	
	# from wilfly to-do backend app, https://github.com/wildfly/quickstart/tree/main/todo-backend
	ENV POSTGRESQL_DATABASE=todos 
	ENV POSTGRESQL_SERVICE_HOST=localhost
	ENV POSTGRESQL_SERVICE_PORT=5432
	ENV POSTGRESQL_USER=todos
	ENV POSTGRESQL_PASSWORD=mysecretpassword
	ENV POSTGRESQL_DATASOURCE=ToDos
	
	ENTRYPOINT [ "sh", "-c", "java $JAVA_OPTS     -jar  /app/todo-backend-bootable.jar"]
	```
1. build image : `docker build -t <defination-your-image-name:wildfly-todo-backend> .`
	- indicate build image named `wildfly-todo-backend` at  `.`  folder. 
	- after successfully built run cmd to check : `docker images`
		```python
		$ docker images
		REPOSITORY                                   TAG       IMAGE ID       CREATED          SIZE
		wildfly-to-do-backend                        latest    3f49601debe0   46 seconds ago   522MB
		registry.access.redhat.com/ubi8/openjdk-11   latest    8d06f878cd4a   41 hours ago     391MB

		```
1. test locally
	1. cmd:  [previous cmd](https://github.com/neokuo0715/monitor-wildfly-example/edit/master/README.md#1-build-wildfly-embedded-application)
	```python
	run successfully 
	2023-03-17 03:32:02.491 UTC [1] LOG:  starting PostgreSQL 15.2 (Debian 15.2-1.pgdg110+1) on x86_64-pc-linux-gnu, compiled by gcc (Debian 10.2.1-6) 10.2.1 20210110, 64-bit
	2023-03-17 03:32:02.495 UTC [1] LOG:  listening on IPv4 address "0.0.0.0", port 5432
	2023-03-17 03:32:02.495 UTC [1] LOG:  listening on IPv6 address "::", port 5432
	2023-03-17 03:32:02.511 UTC [1] LOG:  listening on Unix socket "/var/run/postgresql/.s.PGSQL.5432"
	2023-03-17 03:32:02.534 UTC [64] LOG:  database system was shut down at 2023-03-17 03:32:02 UTC
	2023-03-17 03:32:02.556 UTC [1] LOG:  database system is ready to accept connections
	```
	1. cmd : ` docker run --name todo-backend-1 -e POSTGRESQL_SERVICE_HOST=192.168.99.101 -p 8080:8080 <image-name:wildfly-to-do-backend>`
### 3. push image to your private DockerHub
1. before push, tag your image
	1. cmd : `docker tag <image-name:wildfly-todo-backend> <username:neo1234567>/<image-name:wildfly-todo-backend>:<define-your-tag:v1.0>`
2. push to dockerub
	1. cmd : `docker push <username:neo1234567>/<image-name:>:<tagversion>`
### 4. create cluster by minikube
1. `minikube delete && minikube start --kubernetes-version=v1.24.10 --cpus=4 --memory=6g --bootstrapper=kubeadm --extra-config=kubelet.authentication-token-webhook=true --extra-config=kubelet.authorization-mode=Webhook --extra-config=scheduler.bind-address=0.0.0.0 --extra-config=controller-manager.bind-address=0.0.0.0 --no-vtx-check`
### 5. create & apply secret to grant permission from your private DockerHub
- by [pull-image-private-registry](https://kubernetes.io/docs/tasks/configure-pod-container/pull-image-private-registry/)
1. `docker login`
2. View the `config.json` file: `cat ~/.docker/config.json`
3. create secret yaml, path example `/home/username/.docker/config.json`
	```python
	kubectl create secret generic regcred \
	    --from-file=.dockerconfigjson=<path/to/.docker/config.json> \
	    --type=kubernetes.io/dockerconfigjson
	```
4. check config.json in secret yaml : `kubectl get secret/regcred --output="jsonpath={.data.\.dockerconfigjson}"|base64 --decode`
### 6. create & apply wildfly-deployment 
1. create deploy 、service yaml from folder:wildfly-app same as secret's namespace.
### 7.deploy prometheus and grafana into kubernetes
1. kube-prometheus ( use tool : prometheus-operator ) or prometheus.yaml with configuration yaml ( by configMap )
2. kube-prometheus as example, git clone kube-prometheus project & go into kube-prometheus folder: [[prometheus- monitor activemq#Tool 2 : kube-prometheus [github](https://github.com/prometheus-operator/kube-prometheus kube-prometheus)]]
	```python
	kubectl apply --server-side -f manifests/setup # create CRDS
	kubectl wait \
		--for condition=Established \
		--all CustomResourceDefinition \
		--namespace=monitoring
	kubectl apply -f manifests/ # included: grafana、alertmanagement、nodeExporter... etc
	```
3. change grafana version to 8.3.4 : `kubectl -n monitoring edit deploy/grafana`
4. test : `kubectl port-forward -n wildfly-ex-app svc/<your-app-service> <port-you-define>:<containerPort>`.
	1. `curl <your-service-name>:8080` to list your data
	2. `curl -X POST -H "Content-Type: application/json"  -d '{"title": "This is my first todo item!"}' http://<your-service-name>:8080/ ` to create the info.
