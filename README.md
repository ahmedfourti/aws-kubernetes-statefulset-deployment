AWS-Kubernetes : Deployments VS StatefulSets
============================================


In this repository we will discover Deployments and StatefulSets in Kubernetes Cluster and we will undertstand the difference between them.

**Prerequisite :**

- To follow this repository you need to get Kubernetes Cluster up and running on AWS.  
  You can setup your Kubernetes Cluster here: https://github.com/ahmedfourti/kubernetes-aws  
- Clone this repository.


**The APP**

We will deploy a simple nginx web server with 3 containers and an Elastic Load Balancer which will handle incoming traffic.  


**First method : Deployment**

In the ``deployment.yml`` I have setup 2 sections (called Kind in Kubernetes)  
```Kind: Deployment``` : This is in charge of deploying the pod (one or more containers) on the cluster nodes, in this example I've set the replicas to 3 and added a label ```app: web-nginx```  
```Kind: Service```: This will expose my pods to the world by creating an ELB on AWS and will serve traffic to all pods labeled "web-nginx".  
If for some reasons you want to restrict your ELB access to few IPs, you have to add   
```
   loadBalancerSourceRanges:  
   - XXX.XXX.XXX.XXX/XX 
```    
This must be added on the same level as ```type: LoadBalancer```

Now we can create our app using ```kubectl```  
```kubectl apply -f deployment.yml```  
Wait for 2-3mins and get the ELB hostname from your AWS Console or by typing ```kubectl describe svc my-app```    

Great!!!! We have a high-available nginx up and running !!    
As you can see, with only some lines, we got our nginx up behind a managed ELB, that good....!  
But wait...what will hapened to my data if the container is terminated...? All is lost!! How can I get my volume persistent??  

The solution is..........StatefulSet !  
So the big difference between Deployment and StatefulSet is the volumes ! If your app needs to get a persistent volume, then forget deployment !!  

Deployment ===> Stateless application  
StatefulSet ===> Statefull application  

**Second method : StatefulSet**  

In the statefulset.yml we have 3 sections (I won't discuss about service as it is the same as the first one)  
```kind: StatefulSet```: In this section we define our containers property as we did in the previous part, but we add ```volumeMounts``` and ```volumeClaimTemplates```.  
This first one is used to describe the mount point on each containers and give it a name ```name: my-volume-nginx```.  
The second one is a "claim" which describe the volume that our app will need to work properly. You notice that the name in the metadata section is the exactly the same given to the mount earlier, this is very important. Then we specify a storageClassName (we will explain in it after this). And finally we need the size of our volume.  
That is it for the statefulset section.    

```kin: StorageClass```: This is a very cool feature!! StorageClass will create a type of storage and then this will be consumed by our statefulSet. This is used by Administrators to provide many different type of storage (io1, gp2, sc1...etc) which then will be consumed by developpers when they need to deploy their apps.   
In this example I named my storage class ```my-aws-ebs-gp2``` which is the name that must be specified in the ```volumeClaimTemplate``` and I've set the storage to gp2 which is for general purpose SSD.  

> If you don't set the retain policy, it will defaults to delete (this deletes the volume when the pod is terminated)  

Now we can create our app using ```kubectl```  
```kubectl apply -f statefulset.yml```  
Wait for 2-3mins and login to the AWS Console to see your new volumes created (Make sure to delete them once done)  
Or your can also use ```kubectl get pvc``` and make sure that the status is "Bound"  

```
NAME                                STATUS    VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS     AGE
my-volume-nginx-home-deployment-0   Bound     pvc-62b35f80-ffc9-11e7-bd73-02a96520fb3a   10Gi       RWO            my-aws-ebs-gp2   53s
my-volume-nginx-home-deployment-1   Bound     pvc-6fc6fff0-ffc9-11e7-bd73-02a96520fb3a   10Gi       RWO            my-aws-ebs-gp2   31s
my-volume-nginx-home-deployment-2   Bound     pvc-7c9d47cc-ffc9-11e7-bd73-02a96520fb3a   10Gi       RWO            my-aws-ebs-gp2   9s
```  

You can connect to one of your pods and check that the volume is there (/dev/xvdce) :  

```
root@home-deployment-0:/# df -h
Filesystem      Size  Used Avail Use% Mounted on
overlay         120G  2.7G  113G   3% /
tmpfs           498M     0  498M   0% /dev
tmpfs           498M     0  498M   0% /sys/fs/cgroup
/dev/xvda1      120G  2.7G  113G   3% /etc/hosts
shm              64M     0   64M   0% /dev/shm
/dev/xvdce      9.8G   23M  9.2G   1% /usr/share/nginx/html
tmpfs           498M   12K  498M   1% /run/secrets/kubernetes.io/serviceaccount
tmpfs           498M     0  498M   0% /sys/firmware
```