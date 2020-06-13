# WordPress-On-Minikube-PV-PVC-Secrets-Deployments

# TL; DR

Installing WordPress On Kubernetes.

1. We will create a persistent volume (PV) for both our MySQL & WordPress Pods followed by persistent volume claims (PVC) for the same.
2. Creating A Secret Of MySQL Root Password followed by Deployment of MySQL:5.6 Pod & Attaching MySQL Volume created above.
3. Exposing Deployment with Service Type ClusterIP
4. Creating A Deployment Of WordPress:4.8-apache Pod, Attaching the Volumes & Putting Database Details Using Environment variables.
5. Exposing WordPress Deployment with Service Type NodePort

# Prerequisites

1.Minikube Installed

2.Kubectl configured with Minikube

# Preliminary Steps
1. Start Minikube With Docker Driver
2. minikube start --driver=docker

Once Kubectl is configured, let’s setup our backend first, we will start by creating a Persistent Volume (PV) for MySQL Pod, where all of our Database Configuration & Files will be stored. Since, We are using minikube i.e. All-in-one single node cluster, we are going to use Cluster Node (hostPath) as a Persistent Volume.
Lets configure our Node first by creating directories that will be mounted as a persistent volume on Pods.

To access our node, SSH into the node by
a.minikube ssh
b.sudo mkdir /mnt/sqlvol
c.sudo mkdir /mnt/wpvol
d.exit
e.Listing Volumes

# Persistent Volumes

Our main motive of creating a persistent volume is to store our data outside of pod, so that even if pod crashes or accidently gets deleted, we won’t loose any of our precious data generated by that pod. PV’s as defined is an api-resource provided by kubernetes api-server, which can be represented as piece of storage on Cluster that can be statically provisioned or dynamically provisioned by Cluster administrator.

# So what’s the difference between the two? 

Statically Provisioned Storage :- Storage that is readily available for consumption on the cluster & that needs to be manually created & provisioned by cluster administrator.
Dynamic Provisioned Storage:- This is interesting as this allows storage volumes to be created on-demand i.e. API call to cloud provider to provision new storage volumes & then representing it as persistent volume object is automated. We just need to define the storage class provisioner we need (AWSElasticBlockStore, AzureDisk, iSCSI, NFS, Local etc.) and our persistent volume is created on its own based on the storage class we provide.
To keep our promise of KISS, I am going to use Dynamic Provisioned Storage to create a PV on Cluster Node Storage i.e hostPath
Lets see this in action

# Create a PV with this configuration

\ apiVersion: v1
\ kind: PersistentVolume
metadata:
 name: dbvolume
spec:
 storageClassName: fast
 capacity:
  storage: 2Gi
 accessModes:
 - ReadWriteOnce
 persistentVolumeReclaimPolicy: Recycle
 hostPath:
  path: "/mnt/sqlvol"
  
Understading few parameters used in here:
# storageClassName: This dynamically allotes the type of storage we need that is available on the cluster. We can manually create our storage classes as well.

# accessModes: Different resource providers provide different access modes based on the capabilities (remember every storage class provider have different capabilities). Different accessModes are:

**ReadWriteOnce: 

the volume can be mounted as read-write by a single node
**ReadOnlyMany: 

the volume can be mounted read-only by many nodes
**ReadWriteMany: 

the volume can be mounted as read-write by many nodes
(Since we are using hostPath as a Volume Plugin we are limited to ReadWriteOnce )
**persistentVolumeReclaimPolicy: 

Reclaim policy is what happens to the data stored in PV when PVC is deleted. Different Reclaim Policies Are:
**Retain:

PV will continue to retain even if PVC is deleted.
**Delete: 

PV will be deleted when PVC is deleted.
**Recycle: 

Recycle policy performs a basic scrub (rm -rf /thevolume/*) on the volume and makes it available again for a new claim.
Save YAML file named: dbpv.yaml

# Create a PV onto the cluster :

  **kubectl create -f dbpv.yaml

# Similarly Creating A Persistent Volume for WordPress Pod

apiVersion: v1
kind: PersistentVolume
metadata:
 name: wpvolume
spec:
 storageClassName: slow
 capacity:
  storage: 2Gi
 accessModes:
  - ReadWriteOnce
 persistentVolumeReclaimPolicy: Recycle
 hostPath:
  path: "/mnt/wpvol"

**Save & Create a PV named wppv.yaml

kubectl create -f wppv.yaml
Getting PV’s we just created
kubectl get pv

# Persistent Volume Claims
A Persistent Volume Claim (PVC) as defined is a request for storage to a Persistent Volume. PVC provides an abstraction layer to the underlying storage. In our case, we can ask for storage amount, based on the characteristics of storage from a persistent volume. PVC’s are mostly created by developers to store the data persistently on volumes based on different characteristics and properties of volumes they need.

Lets create a claim for the above volumes we created, Shall We 😀

# PVC for WordPress named wppvc.yaml

apiVersion: v1
kind: PersistentVolumeClaim
metadata:
 name: wpclaim
spec:
 storageClassName: slow
 accessModes:
 - ReadWriteOnce
 resources:
  requests:
   storage: 1Gi
   
kubectl create -f wppvc.yaml
Simlarly, Creating a PVC for MySQL named sqlpvc.yaml

apiVersion: v1
kind: PersistentVolumeClaim
metadata:
 name: sqlclaim
spec:
 storageClassName: fast
 accessModes:
 - ReadWriteOnce
 resources:
  requests:
   storage: 1Gi
   
**kubectl create -f sqlpvc.yaml
Getting PVC’s we just created
**kubectl get pvc

# Secrets
Secrets is an api-resource that provides the security to our sensitive data such as passwords, tokens & credentials. Secrets are useful when we need some credentials that are sensitive but need to be passed in pod without anyone actually seeing the actual information. In our case, since we are using mysql docker image we also need to provide MySQL Root Password. Since its best practice to always hide sensitive information, lets create the same

**kubectl create secret generic sqlpass --from-literal p=pass123

Lets understand the paramters used to create a secret:
–from-literal: This is a type to create a secret in key-value pairs. Another type is –from-file
p is the key for the secret
pass=123 is the password (Highly Secured -,-)
Getting Secret We Just Created
**kubectl get secrets

# Deployments
Deployments is also an api- resource provided by Kubernetes that allows to describe an application’s life cycle, such as which images to use for the app, the number of pods there should be, and the way in which they should be updated, maintaining the number of pods, state of the pod etc. Main puropose of Deployments is to reduce or eliminate down-times during a software update or version update. The scope of deployment to this tutorial is not valid, so lets leave this topic for now. We’ll cover this topic in much greater details in further lessons

Lets get started by creating a Deployment of MySQL named wpdb.yaml . I hope you guys are still doing with me 😀

apiVersion: apps/v1 /n
kind: Deployment
metadata:
 labels:
  app: appdb
 name: appdb
spec:
 replicas: 1
 selector:
  matchLabels:
   app: appdb
 template:
  metadata:
   labels:
    app: appdb
  spec:
   volumes:
    - name: dbvol
      persistentVolumeClaim:
       claimName: sqlclaim
   containers:
    - image: mysql:5.6
      name: mysql
      volumeMounts:
       - name: dbvol
         mountPath: /var/lib/mysql
      env:
       - name: MYSQL_ROOT_PASSWORD
         valueFrom:
          secretKeyRef:
           name: sqlpass
           key: p
Notice how are we attaching the volume using persistentVolumeClaim & attaching it to the pod using volumeMounts
Did you notice, how we passed our secret using secretkeyRef using our sqlpass secrets & using p as a key for our password.
kubectl create -f wpdb.yaml
Exposing MySQL Deployment
kubectl expose deploy appdb --type=ClusterIP --port=3306
–type=ClusterIP: ClusterIP expose the pod internally to the cluster i.e. the scope of ClusterIP is interal network only.
–port=3306: Port at which MySQL Service runs
Similarly Creating Our WordPress Deployment named wpapp.yaml

apiVersion: apps/v1
kind: Deployment
metadata:
 labels:
  app: wordpress
 name: wordpress
spec:
 replicas: 1
 selector:
  matchLabels:
   app: wordpress
 template:
  metadata:
   labels:
    app: wordpress
  spec:
   volumes:
    - name: frontvol
      persistentVolumeClaim:
       claimName: wpclaim
   containers:
    - image: wordpress:4.8-apache
      name: wordpress
      ports:
       - containerPort: 80
      volumeMounts:
       - name: frontvol
         mountPath: /var/www/html/
      env:
       - name: WORDPRESS_DB_HOST 
         value: appdb
       - name: WORDPRESS_DB_PASSWORD
         valueFrom:
          secretKeyRef:
           name: sqlpass
           key: p
Notice how we are creating a volume using persistentVolumeClaim & mounting that volume using volumeMounts under mountPath /var/www/html (Default apache directory)
Notice WORDPRESS_DB_HOST environment variable that is referred to appdb MySQL Service that we just created & WORDPRESS_DB_PASSWORD that is provided by the secret we created.
kubectl create -f wpapp.yaml
Getting The Deployments
kubectl get deploy

Exposing WordPress Deployment
kubectl expose deploy wordpress --type=NodePort --port=80
–type=NodePort: NodePort Exposes A Service To a Node & that service can be accessed by accessing Node IP and port number at which it is exposed.
–port=80: Apache Service Runs By Default on Port 80
Checking The Services
kubectl get services

Find The desired service wordpress & Copy The NodePort
Accessing WordPress
Since minikube provides us with a single node cluster, exposed services can be accessed by getting the IP address of minikube

**minikube ip

**Accessing WordPress From Browser
**Browsing URL at 172.17.0.3:30225
