## Deploy application with kubernetes manifest files

Before start, you should got the **kubeconfig** credentail of CCE cluster and setup the **kubectl** command line tool.

In this example we deployment Jenkins. 
We rename the **kubeconfig** to **kubeconfig-admin.json**.

### 1. create namespace

create a namespace: 'jenkins'

```
kubectl --kubeconfig=kubeconfig-admin.json create ns jenkins
```

```
[root@weather2-72444-sakqs ~]# kubectl --kubeconfig=kubeconfig-admin.json get ns
NAME          STATUS    AGE
aaaaa         Active    5d
default       Active    8d
jenkins       Active    1m
kube-public   Active    8d
kube-system   Active    8d
nginx         Active    1h
nntest        Active    8d

```

### 2. prepare volumes 

- Create PVC
Reference [pvc-evs.yaml](templates/pvc-evs.yaml)

Execute:

```
kubectl --kubeconfig=kubeconfig-admin.json -n jenkins create -f templates/pvc-evs.yaml
```

Query status:

```
[root@weather2-72444-sakqs ~]# kubectl --kubeconfig=kubeconfig-admin.json -n jenkins get pvc
NAME                     STATUS        VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   AGE
cce-evs-jenkins          Bound         pvc-651ef480-3583-11e9-81ed-fa163e2fdd0b   11Gi       RWX            sata           2m

[root@weather2-72444-sakqs ~]# kubectl --kubeconfig=kubeconfig-admin.json -n jenkins get pvc cce-evs-jenkins -o yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  annotations:
    pv.kubernetes.io/bind-completed: "yes"
    pv.kubernetes.io/bound-by-controller: "yes"
    volume.beta.kubernetes.io/storage-class: sata
    volume.beta.kubernetes.io/storage-provisioner: flexvolume-huawei.com/fuxivol
  creationTimestamp: 2019-02-21T02:50:06Z
  enable: true
  finalizers:
  - kubernetes.io/pvc-protection
  labels:
    failure-domain.beta.kubernetes.io/region: ap-southeast-2
    failure-domain.beta.kubernetes.io/zone: ap-southeast-2a
  name: cce-evs-jenkins
  namespace: jenkins
  resourceVersion: "1578643"
  selfLink: /api/v1/namespaces/jenkins/persistentvolumeclaims/cce-evs-jenkins
  uid: 651ef480-3583-11e9-81ed-fa163e2fdd0b
spec:
  accessModes:
  - ReadWriteMany
  resources:
    requests:
      storage: 11Gi
  volumeName: pvc-651ef480-3583-11e9-81ed-fa163e2fdd0b
  volumeNamespace: default
status:
  accessModes:
  - ReadWriteMany
  capacity:
    storage: 11Gi
  phase: Bound

```

### 3. create deployments
Reference [templates/deployment-jenkins.yaml](templates/deployment-jenkins.yaml)
Execute:

```
kubectl --kubeconfig=kubeconfig-admin.json -n jenkins create -f templates/deployment-jenkins.yaml
```

Query:

```
[root@weather2-72444-sakqs ~]# kubectl --kubeconfig=kubeconfig-admin.json -n jenkins get deployments
NAME      DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
jenkins   1         1         1            1           59s
[root@weather2-72444-sakqs ~]# kubectl --kubeconfig=kubeconfig-admin.json -n jenkins get pods
NAME                       READY     STATUS    RESTARTS   AGE
jenkins-8678748768-lvgd7   1/1       Running   0          1m
```

### 4. create tls secret for certificates

Maybe you have no certificate yet, you can reference [this guide](http://t.thought.ink/2016/01/23/%E6%90%AD%E5%BB%BA%E8%87%AA%E5%B7%B1%E7%9A%84CA%E6%9C%8D%E5%8A%A1-OpenSSL-CA-%E5%AE%9E%E6%88%98.html) for creating a self-sign CA and signature a test certificate. After you created certificate and key, you create the secret on the CCE Web UI is simpler.

Reference [templates/sample-certs.yaml](templates/sample-certs.yaml) 

Execute:

```
kubectl --kubeconfig=kubeconfig-admin.json -n jenkins create -f templates/sample-certs.yaml
```

Query:

```
[root@weather2-72444-sakqs ~]# kubectl --kubeconfig=kubeconfig-admin.json -n jenkins get secrets
NAME                  TYPE                                  DATA      AGE
default-secret        kubernetes.io/dockerconfigjson        1         37m
default-token-x9lhn   kubernetes.io/service-account-token   3         37m
paas.elb              cfe/secure-opaque                     2         37m
sample-certs          IngressTLS                            2         24s

```

### 5. expose services

- Expose NodePort service
Reference [templates/svc-nodeport-jenkins.yaml](templates/svc-nodeport-jenkins.yaml)
Exeute:

```
 kubectl --kubeconfig=kubeconfig-admin.json -n jenkins create -f templates/svc-nodeport-jenkins.yaml
```

Query:

```
[root@weather2-72444-sakqs ~]# kubectl --kubeconfig=kubeconfig-admin.json -n jenkins get svc
NAME                TYPE       CLUSTER-IP       EXTERNAL-IP   PORT(S)          AGE
jenkins-intra-vpc   NodePort   10.247.168.169   <none>        8080:30950/TCP   23s
```

- Expose Ingress service to public elb (auto create elb)
Reference [templates/ingress-public-elb-autocreate.yaml](templates/ingress-public-elb-autocreate.yaml)
Execute:

```
kubectl --kubeconfig=kubeconfig-admin.json -n jenkins create -f templates/ingress-public-elb-autocreate.yaml
```

Query:

```
[root@weather2-72444-sakqs ~]# kubectl --kubeconfig=kubeconfig-admin.json -n jenkins get ingress
NAME                     HOSTS                ADDRESS           PORTS     AGE
jenkins-ingress-public   public.jenkins.com   159.138.234.123   80, 443   16s
```

### 6. enable APM
[Sample deployment file](templates/deployment-jenkins-apm-enabled.yaml) with APM enabled.