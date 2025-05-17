# Kubernetes Internal Pod Communication:
## Target setup:
- 2 Containers in the SAME POD that are able to communicate with each other (auth-api + users-api).
- The yaml files configured for this will be in the kubernetes folder under "internal-pod-communication".

## In the user-api->users-app.js we have api call(via axios) to auth-api. So in docker-compose most times we just used the name of the auth-api container and docker would have automatically resolved the name to the ip address of the container. In Kubernetes, since we wanted to put the auth-api+users-api in the same pod they will communicate via localhost, and then in the users-app.js in the api call to auth-api, we need to use the localhost address.
# For that will have an env variable instead of hardcoding the "localhost" as an address. So this would a dynamic value if we would want to run it with docker-compose or in a different environment. (the env variable will be set in the deployment.yaml file).

# Kubernetes Pod-to-Pod Communication:
## Target setup:
- all 3 containers in there own pods (auth-api, users-api, and task-api).
- The yaml files configured for this will be in the kubernetes folder under "pod-to-pod-communication".

## Now since all 3 containers are in different pods, we will want to have a configured service.yaml for each one of them, since pods are ephemeral, the ip address might not be consistent. 
## The service file will be configured differently based on the container and its purpose. 
## we want to establish communication between users-api and auth-api while them being in different pods, and that only the users-api+tasks-api will be exposed to the outside world (while the auth-api will be communicating with the users-api+tasks-api internally). 
- So auth-service.yaml will be configured with port 80 (as exposed in its auth-app.js) in both section - targetPort and port.
- type will be ClusterIP (default) since we want to expose the service internally only. (note that clusterIP is still doing some load balancing by kubernetes, but it is not exposed to the outside world).

- Now we need to provide the auth-api IP address to the users-api here are different ways to do that:

1. Hardcode the IP address of the auth-api clusterIP in the users-deployment.yaml file:
- Go ahead and apply the auth-service.yaml file and then get the ip address that is listed in the output of the command "kubectl get services". You can then copy this ip address and paste it in the users-deployment.yaml file under the env variable value field.
* This method is perfectly fine and the ip wont change once created, but everytime we deploy the yaml files with kubectl apply, we will need to update the users-deployment.yaml file with the new ip address.

2. Use the *service name* instead of manually typing the ip address, kubernetes have by default an environment variable that is created for each service, and it is in the format of your service name - all caps - followed by SERVICE_HOST: "<SERVICE_NAME>_SERVICE_HOST" so here it is: AUTH_SERVICE_SERVICE_HOST.
- Add this AUTH_SERVICE_SERVICE_HOST to users.app.js where we call the auth-ap.
* Make sure to now use the same name in the docker-compose file, so it wont conflict when launching the containers with docker-compose.

3. Use the DNS name of the service instead of the ip address, easiest way and most common way.
- it is an auto generated domain we get by default from kubernetes(core-dns) followed by the name of the service that we have running. And that 

check namespaces:
```bash
kubectl get namespaces
```
- check services:
```bash
kubectl get services
```
- and then create in the format of <my-service-name>.default
- so we will use the cluster default namespace. it will look like:
"auth-service.default" - we will add this to be used inside the env in the users-deployment.yaml file -> and from there it will obviously be used in the users-app.js file -> which will be used in the axios call to auth-api.
* Also make sure to add the "auth-service.default" as env variable in the tasks-deployment.yaml file, cuz the tasks-api will also need to communicate with the auth-api.


# front-end setup:
## up until now we used postman to check api calls. the way we checked it is running minikube service and then using the generated ip+port with /page-name - to check the api call. note that when reaching the page: /tasks you need to provide an auth token with the headers - so under "headers" tab in postman add key "Authorization" with value "bearer abc". Now in order to make the front-end access the tasks pod:
1. add the authorization token in the api-call that is located in the front-end/src/app.js file.
2. copy & paste the address that you get from: "minikube service tasks-service" to the api call that is in the front-end/src/app.js file.(why we didnt simply use the <service-name>.default and instead used the ip address? because the front-end is not running in the same cluster as the tasks-api, so we need to use the ip address that is generated by minikube service).
3. add cors support ('Access-Control-Allow-Origin') in the tasks-app.js file. So when the front-end will try to access the tasks-api it will be allowed to do so.

## After all is setup, you can test it by building the front-end (the Dockerfile that is in the front-end folder), and then running the container with a simple docker command:
```bash
docker run -p 80:80 --rm -d <image-name> # Port 80 is also the port that is exposed in the Dockerfile inside front-end folder.
```
# Deploy the front-end using kubernetes:
1. Build the front-end and push it to the docker hub.
2. Create a frontend-deployment.yaml file and frontend-service.yaml.(the service would include: a type of LoadBalancer, so it will be exposed to the outside world. and port+target port should be set to 80 - as exposed in the nginx.conf+dockerfile).

* read carefully ***
3. Make sure steps 1-3 from the section before are applied. And again the same concept of having the ip address of tasks-api service instead of the <service-name>.default, since the front-end code - app.js - Your React code, even if served from a pod, executes in the browser which is outside the cluster. So we need to use the ip address that is generated by minikube service.


4. Apply the deployment and service files and run "minikube service frontend-service" to get the ip address, paste it in the browser and check if the front-end is working.(if there is already a running deployment you dont want or if the deployment file has changed, you can delete the deployment with "kubectl delete deployment <deployment-name>" and then apply the new one).

# tweaking even more by using a reverse proxy for the front-end:
## The nginx.conf which is also configured in the dockerfile in the front-end folder, is now currently configured to serve the front-end app on port 80. but we can also add the tasks-api to the nginx.conf file, so that the frontend can access the tasks-api without having to use the - hardcoded ip address - of the tasks-api service in the front-end code(app.js) like before.
## remember that the front-end code (app.js) executes in the browser, so we had to use the ip address of the tasks-api service that is generated by minikube service.But since now the front-end app + the nginx.conf is deployed, we can use nginx to access the tasks-api service with the <service-name>.default, cuz the nginx.conf will be executes inside the cluster and it will be able to resolve the <service-name>.default. unlike the front-end code that is executed in the browser.
## So essentially you need to add to nginx.conf this code block:
```nginx
location /api/ {
    proxy_pass http://tasks-service.default:8000/;
}
```
* Any request that is sent to the front-end with the prefix of /api/ will be redirected to the tasks-api service on port 8000(where it listens). So in the front-end code (app.js) you can now use /api/:
```javascript
fetch('/api/tasks', )
```
* So to summerize with this "trick" I can use the reverse proxy to run this code and take advantage of the kubernetes DNS resolver, that executes inside the container - inside the cluster. 
