## Connect Jenkins Pipeline with Kubernetes RBAC
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
kubectl --kubeconfig=kubeconfig-dev.yaml delete pod nginx2-76c669b7c-s2crw
```

You may got the response of forbidden

```
Error from server (Forbidden): pods "nginx2-76c669b7c-s2crw" is forbidden: User "system:serviceaccount:default:dev" cannot delete pods in the namespace "default"
```

6. import the credential(kubeconfig) to Jenkins

