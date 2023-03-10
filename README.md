# Kubernetes

### Basic commands

```bash
# PODS
# create a pod
kubectl run nginx-container --image nginx
kubectl describe pod nginx-container
kubectl get pods -o wide
kubectl delete pod nginx-container
kubectl create -f yamls/01-redis-pod.yaml
# edit pod config
kubectl edit pod redis
#update
kubectl apply -f yamls/01-redis-pod.yaml
# get yaml file from running pod
kubectl get pod nginx-container -o yaml > nginx-pod.yaml

# Replicaset
kubectl get replicasets
kubectl describe replicasets new-replica-set
kubectl apply -f yamls/02-rs.yaml
kubectl delete rs replicaset-1
kubectl scale rs new-replica-set --replicas 5

# Deployment
kubectl get deployments
kubectl describe deployments frontend-deployment

# Namespaces
kubectl get ns
kubectl create ns test
kubectl get pods -n test && kubectl get pods --namespace test
kubectl get pods --all-namespaces

# imperative commands to generate yaml file
kubectl run nginx --image=nginx --dry-run=client -o yaml
kubectl run redis --image redis:alpine --labels tier=db,name=akilan
kubectl expose pod redis --name redis-service --port 6379
kubectl create deployment webapp --image kodekloud/webapp-color --replicas 3
# create a pod and expose it as a single cmd
kubectl run httpd --image httpd:alpine --port 80 --expose
```

### CMD and Entrypoint

```bash
docker run ubuntu # exits immediately
docker run ubuntu sleep 10 # mention entrypoint(binary executable) and args
```

```Dockerfile
# example dockerfile with both entrypoint and cmd
FROM ubuntu
ENTRYPOINT sleep
CMD ["sleep","10"]
```

```bash
docker run myubuntu # sleep 10
docker run myubuntu ls -lrt # overwrite sleep 10
```

```Dockerfile
# example dockerfile with both entrypoint and cmd
FROM ubuntu
ENTRYPOINT sleep
CMD 10
```

```bash
docker run myubuntu # sleep 10
docker run --entrypoint ls myubuntu -lrt  # overwrite entrypoint and cmd
```

### ENV, Configmap and secrets

```bash
kubectl create cm app-config-from-literal --from-literal APP_NAME=my-app
kubectl create cm app-config-from-file --from-file yamls/app.config
kubectl apply -f yamls/05-env-cm-secret.yaml
echo -n mysecret | base64
echo -n bXlzZWNyZXQ= | base64 --decode
```

### SecurityContext

- It is not recommended to run container as root user
- With linuxx capabilities
- Security Context can be in pod level or container level

```bash
docker container run -d --name my-apache --user 1000 --cap-add MAC_ADMIN httpd
# refer 06-security-context.yaml
```

### Resource Requests and Limit

- Set container resource request and limits
- refer yamls/07-resource.yaml

### Service Accounts

- Service account is used to qurey k8s api servers from containers running inside k8s
- from 1.24 onwards k8s wont create secret for each sa
- Token generated from that secret has no expiration date, audience.
- It leads to security issue and scalabilty issue
- so from 1.24 onwards k8s creats sa without secrets
- Each container mounted with projeted volumes which has token

```yaml
volumeMounts:
  - mountPath: /var/run/secrets/kubernetes.io/serviceaccount
    name: kube-api-access-9qchr
    readOnly: true
volumes:
  - name: kube-api-access-9qchr
    projected:
      defaultMode: 420
      sources:
        - serviceAccountToken:
            expirationSeconds: 3607
            path: token
        - configMap:
            items:
              - key: ca.crt
                path: ca.crt
            name: kube-root-ca.crt
        - downwardAPI:
            items:
              - fieldRef:
                  apiVersion: v1
                  fieldPath: metadata.namespace
                path: namespace
```

- login to container and run the below command to list pods in default namespace.
- It will fail as default service account has no permission to list pods

```bash
kubectl apply -f yamls/03-dep.yaml
TOKEN=`cat /var/run/secrets/kubernetes.io/serviceaccount/token`
curl https://kubernetes/api/v1/namespaces/default/pods/ --cacert /var/run/secrets/kubernetes.io/serviceaccount/ca.crt --header "Authorization: Bearer $TOKEN"
```

- so create a new service account and update the pod
- create a pod reader role and role binding to the newly created sa

```bash
kubectl create sa my-sa
kubectl create clusterrole pod-reader --verb=get,list,watch --resource=pods
kubectl create rolebinding pod-reader-binding --clusterrole pod-reader --serviceaccount default:my-sa
kubectl apply -f yamls/08-sa.yaml

```

- now login and test

```bash
kubectl apply -f yamls/03-dep.yaml
TOKEN=`cat /var/run/secrets/kubernetes.io/serviceaccount/token`
curl https://kubernetes/api/v1/namespaces/default/pods/ --cacert /var/run/secrets/kubernetes.io/serviceaccount/ca.crt --header "Authorization: Bearer $TOKEN"
```

### Taints and Tolerations

- taints set on node
- tolerance set on pods
- Taints restricts which can be run on node
- If pods want to run on any particular node use node affinity

```bash
kubectl taint node node01 spray=mortein:NoSchedule #taint
kubectl taint nodes controlplane node-role.kubernetes.io/control-plane:NoSchedule- #untaint
```

### Node selector

```bash
kubectl get nodes --show-labels
kubectl label nodes minikube name=akilan
```

```yaml
nodeSelector:
  name: akilan
```

- to achieve a complex conditions use node affinity and anti affinity

### Node affinity and antiaffinity

```bash
kubectl label nodes node01 color=blue
#refer yamls/10-affinity.yaml
```

### Multi containers - sidecar, initcontainers

- refer yamls/12-sidecar.yaml

### Labels and selectors

```bash
kubectl get pods --selector env=dev
```

### Deployment

```bash
kubectl set image deployment/frontend simple-webap=kodekloud/webapp-color:v2
```

### Jobs and CronJobs

- refer 13-job.yaml & 14-cron-job.yaml

```bash
kubectl create cronjob throw-dice-cron-job --image kodekloud/throw-dice --schedule "30 21 * * *" --dry-run=client -o yaml
```

### Service & Networking policy

- Refer 17-svc.yaml
- Refer 18-nw-policy.yaml

```bash
# ClusterIP, NodePort and LoadBalancer
# NodePort range 30000 to 32767
```

### Ingress controller and Ingress

- Install ingress nginx controller - https://github.com/kubernetes/ingress-nginx/

```bash
kubectl create ingress ingress-test --rule="wear.my-online-store.com/wear*=wear-service:80"
```

### PV, PVC, storage classes

- refer 20-hostpath-volume.yaml, 21-pv.yaml, 22-pvc.yaml and 23-pod-pvc.yaml files under yamls
- With storage classes admin no need to create pv manually. It automatically creates pv
- refer 24-storage-class-pvc.yaml, 26-storage-class.yaml

### Kubernetes Security

- Authentication - Who can access the cluster
- Authorization - What they can do on cluster

- Username and password based authentication was there in k8s till 1.19 but that is no longer supported
- static file - password,user1,u001
- static toke - HGIYUGUIOHGIUGYUIFGIYUIO,user1,u001
- pass the file while starting kube-apiserver with --basic-auth-file

```bash
curl -v -k https://localhost:6443/api/v1/pods -u "user1:password123
```

### Kubeconfig

- 3 important parts of kubeconfig clusters - contexts - users

```bash
kubectl config --help
kubectl config view # view kube config file ~/.kube/config
kubectl config view --kubeconfig=/home/akilan/config #custom config
kubectl config current-context # view current context
kubectl use-context minikube # change the cluster
kubectl config get-clusters # {users,contexts} # get all the clusters
# change default namespace
kubectl config set-context --current --namespace=my-namespace
# switch to different context
kubectl config use-context research
```

### Authorization - Role, Role bindings, Cluster role and Cluster role bindings

```bash
kubectl proxy # proxy kubernetes API to localhost
curl http://localhost:8001/version
# check --authorization-mode=Node,RBAC
kubectl describe pod -n kube-system kube-apiserver-controlplane
kubectl auth can-i create pod --namespace default
kubectl create role pod-manager --verb=get,list,watch,delete,patch,update --resource=pods
kubectl create role deployment-manager --verb=get,list,watch,delete,patch,update --resource=deployments
kubectl auth can-i list pods --namespace default --as dev-user
kubectl create role developer --resource=pods --verb=list,create,delete
kubectl create rolebinding dev-user-binding --role developer --user dev-user --dry-run=client
kubectl create clusterrole storage-admin --resource=persistentvolumes,storageclasses --verb=get,list,create,delete,update
kubectl create clusterrolebinding michelle-storage-admin --clusterrole storage-admin --user michelle
```

### Admission Controller

- An admission controller is a piece of code that intercepts requests to the Kubernetes API server prior to persistence of the object, but after the request is authenticated and authorized.
- Admission controllers may be validating, mutating, or both. Mutating controllers may modify related objects to the requests they admit; validating controllers may not.
- RBAC is good but if we need more control such as
  all resources should have labels,
  container should not run as root,
  Should not use images from docker hub,
  all pods should have liveness and readiness probe etc, we need admission controller
- Better security controls
- If you create a resource under a namespace which doesn't exist throws an error by default
- Enable NamespaceAutoProvision admission controller to create a new one if doesn't exists
- kube-apiserver --enable-admission-plugins=NamespaceLifecycle,LimitRanger ...
- kube-apiserver -h | grep enable-admission-plugins
- kube-apiserver -h | grep disable-admission-plugins
- ps -ef | grep kube-apiserver | grep admission-plugins
- kubectl create secret tls webhook-server-tls --cert /root/keys/webhook-server-tls.crt --key /root/keys/webhook-server-tls.key -n webhook-demo

### Extra

```bash
kubectl api-resources # lists all resources with short names,api versions, namespace scope etc
kubectl proxy 8001&
curl localhost:8001/apis/authorization.k8s.io
kubectl convert -f ingress-old.yaml --output-version networking.k8s.io/v1 > ingress-new.yaml
```
