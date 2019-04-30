# Instalación persistente de MySQL y WordPress en Kubernetes

In this example we will describe how to execute a persistent installation of WordPress and MySQL in Kubernetes. We will use the official images of mysql and wordpress for this installation. (The image of WordPress includes an Apache server). 

We will use the following concepts of Kubernetes: 

* Persistent Volumns (life cicle of disk not linked to Pods).
* Services for allow Pods to locaste each other.
* Load Balancer, a external load balancer to expose the services. 
* Deployments to ensure that the Pods remain operational.
* Secrets to store confidential passwords.

### Requirements

For our example we will need to install Minikube and the Kubectl command on our machine. 

#### KUBECTL

Use the option corresponding to your operating system: [kubectl](https://kubernetes.io/docs/tasks/tools/install-kubectl/#install-kubectl-binary-with-curl)

#### MINIKUBE

For install Minikube we can follow the official documentation [Minikube](https://kubernetes.io/docs/tasks/tools/install-minikube/#install-minikube), but here appear the installation on Windows: 

* First: VT-x/AMD-v vitualization must be enable in BIOS 
* Second: For the installation on Windows, Minikube needs VirtualBox installed in the OS. The version of VirtualBox is not relevant. 
* Third: We will use "chocolatey". To install it:
  * Open a terminal (cmd or powershell) with administrator rights. 
  * If you use cmd execute the next sentence: 
```
@"%SystemRoot%\System32\WindowsPowerShell\v1.0\powershell.exe" -NoProfile -InputFormat None -ExecutionPolicy Bypass -Command "iex ((New-Object System.Net.WebClient).DownloadString('https://chocolatey.org/install.ps1'))" && SET "PATH=%PATH%;%ALLUSERSPROFILE%\chocolatey\bin"
```
  * If you use powershell, first check the execution policy with the command: "Get-ExecutionPolicy" . If the result is: RESTRICTED, then execute :  Set-ExecutionPolicy AllSigned or Set-ExecutionPolicy Bypass -Scope Process . By last:
  
```
Set-ExecutionPolicy Bypass -Scope Process -Force; iex ((New-Object System.Net.WebClient).DownloadString('https://chocolatey.org/install.ps1'))
```

* Fourth: Execute the instalation of minikube using chocolatey. From cmd shell with admin rights execute this command: 

```
choco install minikube
```

* Fifth: open a powershell and execute ```minikube start --v=3``` . This will create a VM instance with the installation of kubernetes locally. 
* Sixth: check if your installation if running properly, for that execute por example: ```kubectl get nodes``` the result must be similar to: 

```
NAME       STATUS   ROLES    AGE   VERSION
minikube   Ready    master   20h   v1.14.0
```


### Installation


* 1) Create the MySQL Password Secret 
  Use a Secret object to store the MySQL password. First create a file (in the same directory as the wordpress sample files) called ```password.txt``` and save your password in it. Make sure to not have a trailing newline at the end of the password. The first tr command will remove the newline if your editor added one. Then, create the Secret object.
  
  ```
  kubectl create secret generic mysql-pass --from-file=password.txt
  ```
* 2) Configure [Persistent Volume](https://kubernetes.io/docs/concepts/storage/persistent-volumes/), we will use Host Path in this example: 

  ```
  kubectl create -f local-volumes.yaml
  ```
  
  Check that have been created properly : 
  
  ```
  Kubectl get pv
  ```  
 
* 3) Deploy MySQL. Now that the persistent disks and secrets are defined, the Kubernetes pods can be launched:

  ```
  kubectl create -f mysql-deployment.yaml
  ```  

Take a look at mysql-deployment.yaml, and note that we've defined a volume mount for ```/var/lib/mysql```, and then created a Persistent Volume Claim that looks for a 20G volume. This claim is satisfied by any volume that meets the requirements, in our case one of the volumes we created above.

Also look at the ```env``` section and see that we specified the password by referencing the secret ```mysql-pass``` that we created above. Secrets can have multiple key:value pairs. Ours has only one key ```password.txt``` which was the name of the file we used to create the secret. The MySQL image sets the database password using the ```MYSQL_ROOT_PASSWORD``` environment variable.

It may take a short period before the new pod reaches the ```Running``` state. List all pods to see the status of this new pod.

```
kubectl get pods
```
```
NAME                              READY   STATUS    RESTARTS   AGE
wordpress-mysql-c747dbfcb-5qvcc   1/1     Running   1          20h
```

Kubernetes logs the stderr and stdout for each pod. Take a look at the logs for a pod by using ```kubectl log```. Copy the pod name from the ```get pods``` command, and then:

```
kubectl logs <pod-name>
```

```
2019-04-25 05:52:28 0 [Warning] TIMESTAMP with implicit DEFAULT value is deprecated. Please use --explicit_defaults_for_timestamp server option (see documentation for more details).
2019-04-25 05:52:28 0 [Note] mysqld (mysqld 5.6.43) starting as process 1 ...
2019-04-25 05:52:28 1 [Note] Plugin 'FEDERATED' is disabled.
2019-04-25 05:52:28 1 [Note] InnoDB: Using atomics to ref count buffer pool pages
2019-04-25 05:52:28 1 [Note] InnoDB: The InnoDB memory heap is disabled
2019-04-25 05:52:28 1 [Note] InnoDB: Mutexes and rw_locks use GCC atomic builtins
2019-04-25 05:52:28 1 [Note] InnoDB: Memory barrier is not used
2019-04-25 05:52:28 1 [Note] InnoDB: Compressed tables use zlib 1.2.11
2019-04-25 05:52:28 1 [Note] InnoDB: Using Linux native AIO
2019-04-25 05:52:28 1 [Note] InnoDB: Using CPU crc32 instructions
2019-04-25 05:52:28 1 [Note] InnoDB: Initializing buffer pool, size = 128.0M
2019-04-25 05:52:28 1 [Note] InnoDB: Completed initialization of buffer pool
2019-04-25 05:52:28 1 [Note] InnoDB: Highest supported file format is Barracuda.
2019-04-25 05:52:28 1 [Note] InnoDB: 128 rollback segment(s) are active.
2019-04-25 05:52:28 1 [Note] InnoDB: Waiting for purge to start
2019-04-25 05:52:28 1 [Note] InnoDB: 5.6.43 started; log sequence number 2973933
2019-04-25 05:52:28 1 [Note] Server hostname (bind-address): '*'; port: 3306
2019-04-25 05:52:28 1 [Note] IPv6 is available.
2019-04-25 05:52:28 1 [Note]   - '::' resolves to '::';
2019-04-25 05:52:28 1 [Note] Server socket created on IP: '::'.
2019-04-25 05:52:28 1 [Warning] Insecure configuration for --pid-file: Location '/var/run/mysqld' in the path is accessible to all OS users. Consider choosing a different directory.
2019-04-25 05:52:28 1 [Warning] 'proxies_priv' entry '@ root@wordpress-mysql-c747dbfcb-5qvcc' ignored in --skip-name-resolve mode.
2019-04-25 05:52:28 1 [Note] Event Scheduler: Loaded 0 events
2019-04-25 05:52:28 1 [Note] mysqld: ready for connections.
Version: '5.6.43'  socket: '/var/run/mysqld/mysqld.sock'  port: 3306  MySQL Community Server (GPL)
```


Also in [mysql-deployment.yaml](mysql-deployment.yaml) we created a service to allow other pods to reach this mysql instance. The name is `wordpress-mysql` which resolves to the pod IP.

Up to this point one Deployment, one Pod, one PVC, one Service, one Endpoint, two PVs, and one Secret have been created, shown below:

```shell
kubectl get deployment,pod,svc,endpoints,pvc -l app=wordpress -o wide 
kubectl get secret mysql-pass 
kubectl get pv
```

* 4) Deploy WordPress: 

Next deploy WordPress using [wordpress-deployment.yaml](wordpress-deployment.yaml):

```shell
kubectl create -f wordpress-deployment.yaml
```

Here we are using many of the same features, such as a volume claim for persistent storage and a secret for the password.

The [WordPress image](https://hub.docker.com/_/wordpress/) accepts the database hostname through the environment variable `WORDPRESS_DB_HOST`. We set the env value to the name of the MySQL service we created: `wordpress-mysql`.

The WordPress service has the setting `type: LoadBalancer`.  This will set up the wordpress service behind an external IP.

```shell
kubectl get services wordpress
```

```shell
NAME        TYPE           CLUSTER-IP     EXTERNAL-IP   PORT(S)        AGE
wordpress   LoadBalancer   10.105.81.79   <pending>     80:30224/TCP   20h
```

* 5) The last step is expose the service:

```shell
minikube service wordpress
```
  
  You should see the familiar WordPress init page.


## Take down and restart your blog

Set up your WordPress blog and play around with it a bit. Then, take down its pods and bring them back up again. Because you used persistent disks, your blog state will be preserved.

All of the resources are labeled with `app=wordpress`, so you can easily bring them down using a label selector:

```shell
kubectl delete deployment,service -l app=wordpress
kubectl delete secret mysql-pass
```

Later, re-creating the resources with the original commands will pick up the original disks with all your data intact. Because we did not delete the PV Claims, no other pods in the cluster could claim them after we deleted our pods. Keeping the PV Claims also ensured recreating the Pods did not cause the PD to switch Pods.

If you are ready to release your persistent volumes and the data on them, run:

```shell
kubectl delete pvc -l app=wordpress
```

And then delete the volume objects themselves:

```shell
kubectl delete pv local-pv-1 local-pv-2
```

