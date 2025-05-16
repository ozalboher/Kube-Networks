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
- use the auto generated domains we get by default from kubernetes(core-dns). and the name of the service.

check namespaces:
```bash
kubectl get namespaces
```
- check services:
```bash
kubectl get services
```
- and then create in the format of <my-service-name>.<namespace>
- so we will use the cluster default namespace. it will look like:
"auth-service.default" - we will add this to be used inside the env in the users-deployment.yaml file -> and from there it will obviously be used in the users-app.js file -> which will be used in the axios call to auth-api.
* Also make sure to add the "auth-service.default" as env variable in the tasks-deployment.yaml file, cuz the tasks-api will also need to communicate with the auth-api.