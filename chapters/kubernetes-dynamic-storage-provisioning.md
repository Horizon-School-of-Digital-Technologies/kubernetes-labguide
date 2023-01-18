# Dynamic Storage Provisioning

This tutorial explains how kubernetes storage works and the complete workflow for the dynamic provisioning. The topics include

  * Storage Classes
  * PersistentVolumeClaim
  * persistentVolume
  * Provisioner

Pre Reading :

  * [Kubernetes Storage Concepts](https://youtu.be/hqE5c5pyfrk?t=461)
  * [Storage Classes](https://kubernetes.io/docs/concepts/storage/storage-classes/)


## Concepts

![kubernetes Storage Concepts](images/storage_mindmap.png)

## Deploy Database with a Persistent Volume Claim

Lets begin by redeploying the db deployment, this time by configuring it to refer to the persistentVolumeClaim

`file: db-deploy-pvc.yaml`

```
...
spec:
   containers:
   - image: postgres:9.4
     imagePullPolicy: Always
     name: db
     ports:
     - containerPort: 5432
       protocol: TCP
     #mount db-vol to postgres data path
     volumeMounts:
     - name: db-vol
       mountPath: /var/lib/postgresql/data
   #create a volume with pvc
   volumes:
   - name: db-vol
     persistentVolumeClaim:
       claimName: db-pvc
```

Apply *db-deploy-pcv.yaml*  as

```
kubectl apply -f db-deploy-pvc.yaml
```

To monitor resources for this lab, open a new terminal and start watching for relevant objecting using the following command. 

```

watch kubectl get pods,pvc,pv,storageclasses 
```
We will call the terminal where you are running the above command as your **Monitoring Screen**. 

  * Observe and note if the pod for *db* is launched.
  * What state is it in ? why?
  * Has the persistentVolumeClaim been bound to a persistentVolume ? Why?


## Creating a Persistent Volume Claim

switch to project directory

```
cd k8s-code/projects/instavote/dev/
```

Create the following file with the specs below

`file: db-pvc.yaml`

```
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: db-pvc
spec:
  accessModes:
    - ReadWriteOnce
  volumeMode: Filesystem
  resources:
    requests:
      storage: 2Gi
  storageClassName: gp2

```

`if you are using your own cluster, make sure to use the right storage class.`

create the Persistent Volume Claim and validate

```
kubectl get pvc


kubectl apply -f db-pvc.yaml

kubectl get pvc,pv

```

  * Is persistentVolumeClaim created ?  Why ?
  * Is persistentVolume created ?  Why ?
  * Is the persistentVolumeClaim bound with a persistentVolume ?


### Validate

Now, observe the output of  the following commands,

```
kubectl get pvc,pv
kubectl get pods
```

  * Do you see pvc bound to pv ?
  * Do you see the pod for db running ?

Observe the dynamic provisioning, go to the host which is running nfs provisioner and look inside */srv* path to find the provisioned volume.


### Troubleshooting NFS Provisioner 

If you do not see the volume been provisioned despite creating storageclass, it may be due to a problem with API server configurations.  

To find out if thats the case, check the logs for **nfs-provisioner** first. 


```
I0319 16:00:21.091140       1 controller.go:1052] scheduleOperation[lock-provision-instavote/db-pvc[06780d6b-a6d0-4701-bd72-faaa507b0e5a]]
I0319 16:00:21.106978       1 leaderelection.go:154] attempting to acquire leader lease...
I0319 16:00:21.121501       1 leaderelection.go:176] successfully acquired lease to provision for pvc instavote/db-pvc
I0319 16:00:21.121686       1 controller.go:1052] scheduleOperation[provision-instavote/db-pvc[06780d6b-a6d0-4701-bd72-faaa507b0e5a]]
E0319 16:00:21.155945       1 controller.go:751] Unexpected error getting claim reference to claim "instavote/db-pvc": selfLink was empty, can't make reference

``` 
If you see messages similar to above with error string containing `selfLink was empty`, you could fix it by updating API Server configurations.  

To update API server configurations, update the pod spec that launches the API Server. 

On the master node edit the file : ```/etc/kubernetes/manifests/kube-apiserver.yaml```

and add ```- --feature-gates=RemoveSelfLink=false``` option the the command. An example is as below. 

```
  - command:
    - kube-apiserver
    - --advertise-address=143.198.50.211
    - --allow-privileged=true
    - --feature-gates=RemoveSelfLink=false
```

Once you make this change, save the file, wait for a minute or so and you shall see the volume provisioned. 

## Nano Project [Optional Exercise]

Similar to postgres which mounts the data at /var/lib/postgresql/data and consumes it to store the database files, Redis creates and stores the file at **/data** path.  Your task is to have a nfs volume of size **200Mi** created and mounted at /data for the redis container.

You could follow these steps to complete this task

  * create a pvc by name **redis**
  * create a volume in the pod spec with type persistentVolumeClaim. Call is **redis-data**
  * add volumeMounts to the container spec (part of the same deployment file) running redis and have it mount **redis-data** volume created in the pod spec above.





#### Summary

In this lab, you not only setup dynamic provisioning using NFS, but also learnt about statefulsets as well as rbac policies applied to the nfs provisioner.
