# Goal 

We show how to install Kasten when you have no storage provisioner at all on your cluster.

## Use case

Some cluster may not have storage provisioner at all but still need Kasten to protect their applications and eventually work with blueprint to backup internal or external databases.

However Kasten expect that a storage class is present on the cluster so that it can create his own PVC for the inventory databases (the catalog) or run the jobs queue, prometheus, grafana or logging.

## Kasten don't need a very high storage performance for it's internal functionning

Kasten don't need a very high storage performance for it's internal functionning, a simple internal NFS server will let us create a NFS provisionner on which Kasten will be able to run.

## install the NFS provisionner 

Note : if you don't have a NFS server available you can create one on kubernetes itself check this [section](#create-a-nfs-sever-on-kubernetes-itself).


Set up this two variables 
```
nfsIp=<Your NFS IP>
nfsPath=<Your NFS path on the server>
```

and install the NFS Sub dir provisionner
```
helm repo add nfs-subdir-external-provisioner https://kubernetes-sigs.github.io/nfs-subdir-external-provisioner/
helm upgrade --install -n nfs-storage --create-namespace nfs-subdir-external-provisioner nfs-subdir-external-provisioner/nfs-subdir-external-provisioner \
    --set nfs.server=$nfsIp \
    --set nfs.path=$nfsPath \
    --set storageClass.name=k10-storage \
    --set storageClass.archiveOnDelete=false
```

## install Kasten with the k10-storage storage class

You now have a storage class k10-storage that you can use for installing kasten all you you have to do is specify the storage class k10-storage

```
helm repo add kasten https://charts.kasten.io/
helm repo update 
helm upgrade k10 kasten/k10 --create-namespace --install -n kasten-io \
   --set global.persistence.storageClass=k10-storage
```


## Create a NFS Sever on kubernetes itself

*For most of the user this step can be skipped because there is always an NFS server available on the data center.*

But if you don't have it then you can create a nfs server directly on the cluster using  hostpath or local storage. 

Using local storage is more complex has [we need to configure disk on the worker nodes](https://github.com/kubernetes-sigs/sig-storage-local-static-provisioner/blob/master/docs/operations.md) and let the local provisionner handle the creation of PV that PVC will bound later. 

We want something quick for this so let's use hospath for our nfs server and use a node name to make sure this pod is alway scheduled on the same node. 


```
nodeName=<your node name>
cat<<EOF | kubectl create -f -
apiVersion: v1
kind: Namespace
metadata:
  name: nfs-storage  
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nfs-server
  namespace: nfs-storage
spec:
  selector:
    matchLabels:
      app: nfs-server
  template:
    metadata:
      labels:
        app: nfs-server
    spec:
      nodeName: $nodeName
      containers:
      - name: nfs-server
        image: k8s.gcr.io/volume-nfs:0.8
        ports:
        - name: nfs
          containerPort: 2049
        - name: mountd
          containerPort: 20048
        - name: rpcbind
          containerPort: 111
        securityContext:
          privileged: true
        volumeMounts:
        - name: storage
          mountPath: /exports
      volumes:
      - name: storage
        hostPath:
          path: /data/nfs 
          type: DirectoryOrCreate 
---
apiVersion: v1
kind: Service
metadata:
  name: nfs-server
  namespace: nfs-storage
spec:
  ports:
  - name: nfs
    port: 2049
  - name: mountd
    port: 20048
  - name: rpcbind
    port: 111
  selector:
    app: nfs-server
EOF
```

Check the nfs server is running 
```
kubectl get po -n nfs-storage 
```

You should get an output like this 
```
NAME                                               READY   STATUS    RESTARTS   AGE
nfs-server-6b645cf6fd-ttv9b                        1/1     Running   0          4h49m
```

Now you can set up the ip of the nfs server and your path will be simply "/"
```
nfsIp=$(kubectl get svc -n nfs-storage nfs-server -o jsonpath='{.spec.clusterIP}')
nfsPath="/"
```
