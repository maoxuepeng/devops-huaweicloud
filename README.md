# devops-huaweicloud
DevOps on HuaweiCloud

### 1. Build a docker image based on Jenkins and add libltdl7 library for docker exec
reference the [Dockerfile-jenkins-2.150.3.libltdl7](Dockerfile-jenkins-2.150.3.libltdl7), build it and push to SWR.
1. create dockerfile
2. build docker image
docker build -t jenkins:2.150.3-libldt7 -f Dockerfile .
3. tag the image
docker tag ed92377e7e1d swr.ap-southeast-2.myhuaweicloud.com/xxx/jenkins:2.150.3-libldt7
4. login to swr
5. push image
docker push swr.ap-southeast-2.myhuaweicloud.com/xxx/jenkins:2.150.3-libldt7

### 2. Install and config Jenkins
1. Jenins image jenkins-2.150.3.libltdl7
2. Enable privileged container
3. Volume mounts
    - Persistent storage(EVS) mount on /var/jenkins_home
    - Local path /var/run/docker.sock to container path /var/run/docker.sock
    - Local path /usr/bin/docker to container path /usr/bin/docker
4. expose 8080 port to external access
5. Login Jenkins, setup first user, install default plugins
6. Install **Kubernetes Cli** and **[Kubernetes Continuous Deploy](https://wiki.jenkins.io/display/JENKINS/Kubernetes+Continuous+Deploy+Plugin)** plugin

### 3. Get SWR credential
1. Prepare the SWR credential from [this guide](https://support-intl.huaweicloud.com/usermanual-swr/swr_01_1000.html)
2. summary of the setps:
- got the AK/SK first
- open shell, set ```AK=<your access key>```, ```SK=<your secret key>```
- execute ```printf "$AK" | openssl dgst -binary -sha256 -hmac "$SK" | od -An -vtx1 | sed 's/[ \n]//g' | sed 'N;s/\n//'```, you will get your login key
- replace you information into this command: ```docker login -u [Name of the regional project]@[AK] -p [Login key] [Image repository address]```

### 4. Get CCE credential
1. go to Resource Management -> Cluster Manageent -> Select your destination cluster -> Click More and Kubectl -> On the page click link under "Download the kubectl configuration file"

### 5. Import CCE credential to Jenkins
create a kubeconfig credential in jenkins

### 6. create a pipeline job
reference the pipnline script [jenkins-sample.pipeline](jenkins-sample.pipeline)

### 7. execute the pipeline job
execute and check the deployment on CCE.

## Advanced
### 1. Connect Jenkins Pipeline with Kubernetes RBAC
There is an example show how to control user permission with RBAC. The developer can create/upgrade Deployments/Pods in certain namesapce, but can **NOT DELETE** Deployments/Pods in certain namesapce.

**Before start, get the admin kubeconfig file from CCE cluster, rename to kubeconfig-admin.json**

1. create a ServiceAccount in Kubernetes under a namespace
Create service account name **dev** under **default** namespace.
```
kubectl --kubeconfig=kubeconfig-admin.json -n default create sa dev
```
2. define the permission in Kubernetes Role file
Create Kubernetes Role object by YAML.
```
cat <<EOF > dev-user-role.yml
kind: Role
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  namespace: default   #should replace to your own namesapce
  name: dev-user
rules:
- apiGroups: ["*"]
  resources: ["deployments", "pods", "pods/log"]  # which resources allowed
  verbs: ["get", "watch", "list", "update", "create"] # which operations allowed
EOF
```

Create the Role in Kubernetes cluster
```
kubectl --kubeconfig=kubeconfig-admin.json create -f dev-user-role.yml
```

3. bind the role with service account
```
kubectl --kubeconfig=kubeconfig-admin.json create rolebinding dev-view \
    --role=dev-user \
    --serviceaccount=default:dev \
    --namespace=default
```
4. create Kubernetes credential(kubeconfig) for the service account
- Create kubeconfig file for dev role

```
API_SERVER="https://192.168.0.168:5443"  #should replace to your own kubernetes API server address
SECRET=$(kubectl --kubeconfig=kubeconfig-admin.json -n default get sa dev -o go-template='{{range .secrets}}{{.name}}{{end}}')
CA_CERT=$(kubectl --kubeconfig=kubeconfig-admin.json -n default get secret ${SECRET} -o yaml | awk '/ca.crt:/{print $2}')
cat <<EOF > kubeconfig-dev.yaml
apiVersion: v1
kind: Config
clusters:
- cluster:
    certificate-authority-data: $CA_CERT
    server: $API_SERVER
  name: weather2  # replace to your cluster name
EOF

```

- Set credentials(Token) into the kubeconfig

```
TOKEN=$(kubectl --kubeconfig=kubeconfig-admin.json -n default get secret ${SECRET} -o go-template='{{.data.token}}')
kubectl --kubeconfig=kubeconfig-dev.yaml config set-credentials dev-user \
    --token=`echo ${TOKEN} | base64 -d` 

```

- Set dev credentail as default context into the kubeconfig

```
kubectl --kubeconfig=kubeconfig-dev.yaml config set-context default \
    --cluster=weather2 \
    --user=dev-user 

```

- Switch to dev context for kubectl
```
kubectl --kubeconfig=kubeconfig-dev.yaml config use-context default

```

5. verify the permission
Try to delete some pod

```

```

6. import the credential(kubeconfig) to Jenkins

