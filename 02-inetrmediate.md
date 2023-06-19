
# Lab - Intermediate Kubernetes Concepts

Estimated Duration: 60 minutes

Exercices
- Verify Kubernetes Cluster
- Exercise: Working with Multi-Container Pods, Ephemeral Volumes and ConfigMaps
- Exercise: Working with Persistent Volumes, Persistent Volume Claims and Secrets
- Exercise: Using Ingress Resources and Ingress Controller to Control External Access
- Shutdown or Delete the Cluster (cloud provider only)

### Exercise: Verify Kubernetes Cluster

In this exercise you will verify the Kubernetes cluster.

Open a Windows Terminal window (defaults to PowerShell).
Windows Terminal allows you to open tabbed command terminals.

The following example shows how to connect the cluster to your local client machine for AKS (similar approaches should be available for other providers):

``` PS
$clusterName = 'your cluster'
$rg = 'Azure RG'
az aks get-credentials --name $clusterName --resource-group $rg
```

Confirm the connection to the cluster.

``` PS
kubectl get nodes
```

This should return the list of nodes.

### Exercise: Working with Multi-Container Pods, EphemeralVolumes and ConfigMaps

This exercise shows how multiple containers within the same pod can use volumes to communicate with each other and how you can useConfigMaps to mount settings files into containers.
The example Deployment below configures a MySQL instance to write its activities to a log file. Another container reads that log file, andin a production setting, would send the contents to an external log aggrigator. In this example, the 2nd container just outputs the tail ofthe log file to the console.

**Task 1 - Review and deploy ConfigMaps**

Open a Windows Terminal window (defaults to PowerShell).

Change current folder to *02-intermediate*

``` PS
cd 02-intermediate
```

Review the mysql-initdb-cm.yaml file.

``` PS
cat mysql-initdb-cm.yaml
```

This sql script will be used to initialize the database.

``` yaml
apiVersion: v1
kind: ConfigMap
metadata: 
  name: mysql-initdb-cm
data: initdb.sql: | 
  CREATE DATABASE sample; 
  USE sample; 
  CREATE TABLE friends (id INT, name VARCHAR(256), age INT, gender VARCHAR(3));
  INSERT INTO friends VALUES (1, 'John Smith', 32, 'm'); 
  INSERT INTO friends VALUES (2, 'Lilian Worksmith', 29, 'f'); 
  INSERT INTO friends VALUES (3, 'Michael Rupert', 27, 'm');
```

When the ConfigMap is mounted in a container, it will create a file called initdb.sql.

Deploy the ConfigMap.

``` PS
kubectl apply -f mysql-initdb-cm.yaml
```

Configure MySql to generate activity logs and save them to a file. 
Review the contents of mysql-cnf-cm.yaml. This file will replace the default my.cnf config file in the container.

``` yaml
apiVersion: v1
kind: ConfigMap
metadata: 
  name: mysql-cnf-cm
data: my.cnf: | 
  !includedir /etc/mysql/conf.d/ 
  !includedir /etc/mysql/mysql.conf.d/ 
  
  [mysqld] 
  #Set General Log 
  general_log = on 
  general_log_file=/usr/log/general.log
```

Deploy the ConfigMap.

``` PS
kubectl apply -f mysql-cnf-cm.yaml
```

**Task 2 - Review and deploy multi-container Deployment**

Review the Volumes section of mysql-dep.yaml.

``` yaml
volumes:
 - name: mysql-initdb 
   configMap: 
     name: mysql-initdb-cm
 - name: mysql-cnf 
   configMap: 
     name: mysql-cnf-cm
 - name: mysql-log 
   emptyDir: {}
```

Notice that both ConfigMaps are being mapped to volumes and an empty directory is being created for the logs.

Review the volumeMounts section of mysql-dep.yaml.

``` yaml
volumeMounts:
 - name: mysql-initdb 
   mountPath: /docker-entrypoint-initdb.d
 - name: mysql-cnf 
   mountPath: /etc/mysql/my.cnf 
   subPath: my.cnf
 - name: mysql-log 
   mountPath: /usr/log/
```

The init file is mapped to the known MySql folder. Anything placed is that folder is executed by MySql during it startup.

The config file is mapped to the MySql config folder. However, there are other config files already in that folder that MySql needs toaccess. As a result, replacing the entire folder with the contents of the ConfigMap will prevent MySql from working, because the otherconfig files won't be there.

That's where the subPath property comes into play. It allow you to mount only a subset of your volume (in this case only a single file),leaving the exiting content in place at the destination.

The final mount points to the empty directory created to hold the logs.

Review the second container definition.

``` yaml
- name: logreader 
  image: busybox 
  command:
  - "/bin/sh" 
  args:
  - "-c"
  - "tail -f /usr/log/general.log;"
  volumeMounts:
  - name: mysql-log 
    mountPath: /usr/log/
```

Notice that the same empty directory is mounted to this container and that the container simply executes a tail command, outputtingthe last line in the file.

Apply the deployment.

``` PS
kubectl apply -f mysql-dep.yaml
```

**Task 3 - Confirm communications between containers**

Get list of Pods.

``` PS
kubectl get pods
```

Once the Pod is running, look at the log of the second container.

``` PS
kubectl logs -c logreader -f "name of mysql-dep-xxxxx pod"
```

Open another shell window and execute into the first container

``` PS
kubectl exec -it -c mysql "name of mysql-dep-xxxxx pod" -- bash
```

In the shell, type mysql to enter the MySql console.

``` sh
mysql
```

Type the following commands in the mysql> prompt:
``` SQL
use sample;
select * from friends;
```

Switch to the other window and verify that the second container is showing the log.

Exit out of the MySql console.

``` sh
exit;
```

Exit out of the container.
``` sh
exit
```

### Exercise: Working with Persistent Volumes, Persistent VolumeClaims and Secrets

This exercise shows an example of using a secret to store the password needed by an Azure File Share. You will first create the Azure FileShare, get its Access Key and then configure a secret to use that access key to connect a volume to a Pod.

**Task 1 - Create an Azure Storage Account and an Azure File Share (clud provider only)**

This example uses Azure to create a storage account and mount the file share as PVC in Kubernetes.

Open a Windows Terminal window (defaults to PowerShell).

Login to Azure.
``` PS
az login
az account set --subscription "your subscription"

Define variables.

``` PS
$INITIALS="abc"
$YOUR_INITIALS="$($INITIALS)".ToLower()
$AKS_RESOURCE_GROUP="azure-$($INITIALS)-rg"
$STORAGE_ACCOUNT_NAME="sa$($YOUR_INITIALS)"
$SHARE_NAME="share$($YOUR_INITIALS)"
$LOCATION="your region"
```

Create the Azure Storage Account.

``` PS
az group create --location $LOCATION --resource-group $AKS_RESOURCE_GROUP
az storage account create --name $STORAGE_ACCOUNT_NAME `
    --resource-group $AKS_RESOURCE_GROUP `
    --sku Standard_LRS
```

Create the Azure File Share.

``` PS
az storage share create --name $SHARE_NAME `
    --connection-string $(az storage account show-connection-string `
    --name $STORAGE_ACCOUNT_NAME `
    --resource-group $AKS_RESOURCE_GROUP -o tsv)
```

The Account Name and Account Key will be echoed to the screen for reference.

``` PS
$STORAGE_KEY=$(az storage account keys list `
    --resource-group $AKS_RESOURCE_GROUP `
    --account-name $STORAGE_ACCOUNT_NAME `
    --query "[0].value" -o tsv)
$STORAGE_ACCOUNT_NAME$STORAGE_KEY
```

**Task 2 - Create a Namespace for this lab**

All the objects related to this lab will be self-contained in a Namespace. When you're done, delete the Namespace and all the objects itcontains will also be deleted. The the Persistent Volume, which is scoped across the entire cluster, will remain.

Create a new Namespace.

``` PS
kubectl create ns lab-volume
```

Set the new Namespace as the current namespace, so it doesn't have to be specified in future commands.

``` PS
kubectl config set-context --current --namespace lab-volume
```

**Task 3 - Create a Secret imperatively**

The Account Name and Account Key need to be saved into a Kubernetes Secret object. The best way to do this is to create theSecret imperatively:

``` PS
kubectl create secret generic azure-secret `
    --from-literal=azurestorageaccountname=$STORAGE_ACCOUNT_NAME `      --from-literal=azurestorageaccountkey=$STORAGE_KEY `
    --dry-run=client `
    --namespace lab-volume `
    -o yaml > azure-secret.yaml
```

A YAML file is generated with all the information needed to create the secret. Review the new file.
cat azure-secret.yaml

> **_NOTE:_** It's best to create the file first and then apply it so you can repeat the process if needed.

Apply the secret in the default namespace. Since the Persistent Volume is not a namespaced resource, it will expect to find thesecret in the default namespace.

``` PS
kubectl apply -f azure-secret.yaml -n lab-volume
```

**Task 4 - Create a Persistent Volume and a Persistent Volume Claim**

The first step to accessing an Azure File Share is creating a PersistentVolume object that connects to that share.
Review the value of SHARE_NAME created in Task 1 above.

``` PS
echo $SHARE_NAME
```

> **_CRITICAL STEP:_** Review the contents of azurefile-pv.yaml. Make sure the shareName property matches the value of SHARE_NAME.

Also notice there's a label associated with the persistent volume:

When the file is ready, apply it.

``` PS
kubectl apply -f azurefile-pv.yaml
```

Make sure the PersistentVolume was created and has a status of Available.

``` PS
kubectl get pv
```

Review the contents of azurefile-pvc.yaml. Notice that the label selector is set to look for the persistent volume:

Also the label selector is set to find the PV.

Apply the Persistent Volume Claim.

``` PS
kubectl apply -f azurefile-pvc.yaml
```

Verify that the Persistent Volume Claim is bound to the Persistent Volume.

``` PS
kubectl get pvc
```

Review the PersistentVolume again

``` PS
kubectl get pv
```

Notice that the PVC claimed the PV.

**Task 5 - Use the Persistent Volume Claim in a Pod**

Now that the connection is configured to the Azure File Share, create a Pod to use the Persistent Volume Claim.

Review the contents of pvc-pod.yaml. Notice how the persistentVolumeClaim is being used and mountPath of thevolumeMount.

Apply the Pod.

``` PS
kubectl apply -f pvc-pod.yaml
```

Confirm the Pod is running. It might take a few minutes for the Pod to connect to the PVC.

``` PS
kubectl get pod pvc-pod
```

Once the Pod is running, shell into the Pod.

``` PS
kubectl exec -it pvc-pod -- bash
```

Change into the shared folder, create a file and confirm it's there.

``` sh
cd /sharedfoldertouch abc123.txtls -l
```
	
Open the Azure Portal in a browser.

Navigate to the storage account created earlier (as defined in STORAGE_ACCOUNT_NAME).

Click on the File shares selection.

Click on the file share name.

You should see the file you created in the container. Click the Upload button and upload a file from your hard drive to theshare.

Get a list of files in the shared folder again.

``` sh
ls -l
```

Both files should be in the shared folder.

This confirms that the sharedfolder is mapped to the Azure File Share.

Exit the container.

``` sh
exit
```

**Task 6 - Set the current Namespace back to default**

Set the current namespace back to default so all of these objects stay separate from the other labs.

``` PS
kubectl config set-context --current --namespace default
```

OPTIONAL: If you prefer, you can delete the namespace. This will delete all the objects (secrets, persistent volume claim, pods)from the cluster. The only object that will remain will be the Persistent Volume, because it's not tied to any specific namespace.

``` PS
kubectl delete ns lab-volume
```

### Exercise: Using Ingress Resources and Ingress Controller tocontrol external access (cloud provider only)

Ingress resources and 3rd-party Ingress Controllers can be used to centrally control external access to the cluster.

**Task 1 - Install NGinx Ingress Controller**

Change current folder to C:\k8s\labs\Module3.

``` PS
cd 02-intermediate
```

Install Nginx Ingress Controller using Helm:

``` PS
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo update
helm upgrade --install ingress-nginx ingress-nginx `
    --repo https://kubernetes.github.io/ingress-nginx `
    --namespace ingress-nginx --create-namespace `
    --set controller.nodeSelector."kubernetes\.io/os"=linux `
    --set defaultBackend.nodeSelector."kubernetes\.io/os"=linux `
    --set controller.service.externalTrafficPolicy=Local `
    --set defaultBackend.image.image=defaultbackend-amd64 `
    --set defaultBackend.image.tag=1.5
```

Wait a few minutes to allow the Nginx Load Balancer service to aquire an external IP address.

Query the services in the ingress-nginx namespace.

``` PS
kubectl get svc -n ingress-nginx
```

Get the EXTERNAL IP of the Load Balancer:

``` PS
NAME TYPE CLUSTER-IP EXTERNAL-IP PORT(S) AGE
service/ingress-nginx-controller LoadBalancer 10.0.240.47 52.146.66.160 80:32616/TCP,443:31149/TCP 108s
service/ingress-nginx-controller-admission ClusterIP 10.0.244.155 <none> 443/TCP 108s
```

**Task 2 - Define a Default Backend**

Open the browser and enter the exnteral IP of the Ingress controller by itself: http://"external ip"

This is the default NGinx Ingress Controller page.

Review the contents of C:\k8s\labs\Module3\default-backend.yaml file in an editor. 
Notice the special property to specify the default backend. There's no path property needed.

``` yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata: 
  name: default-ingress-backend
spec: 
  defaultBackend: 
    service: 
      name: default-svc 
      port: 
        number: 8100
```

Apply the Default Backend deployment, service and ingress resource to the default namespace (not dev).

``` PS
kubectl apply -f default-dep.yaml -n defaultkubectl apply -f default-svc.yaml -n defaultkubectl apply -f default-backend.yaml -n default
```

Open the browser and enter the exnteral IP of the Ingress controller by itself: http://"external ip"

**Task 3 - Install Deployments, Services and Ingress Resource in a separate namespace**

Create the dev namespace

``` PS
kubectl create ns dev
```

Change current folder to C:\k8s\labs\Module3.

``` PS
cd 02-intermediate
```

Apply the deployments and services.

``` PS
kubectl apply -f blue-dep.yaml -f blue-svc.yaml -n devkubectl apply -f red-dep.yaml -f red-svc.yaml -n dev
```

Review the contents of C:\k8s\labs\Module3\colors-ingress.yaml file in an editor. Notice the path setting to route traffic tothe correct service.

``` yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata: 
  annotations: 
    kubernetes.io/ingress.class: nginx 
    nginx.ingress.kubernetes.io/rewrite-target: /$1 \
  name: colors-ingress
spec: 
  rules:
  - http: 
      paths:
      - path: /blue/(.*) 
        pathType: Prefix 
        backend: 
          service: 
            name: blue-svc 
            port: 
              number: 8100
      - path: /red/(.*) 
        pathType: Prefix 
        backend: 
          service: 
            name: red-svc 
            port: 
              number: 8100
```

Apply the Ingress resource.

``` PS
kubectl apply -f colors-ingress.yaml -n dev
```

Verify the ingress resource was created.

``` PS
kubectl get ing -n dev
```

Returns:

``` PS
NAME CLASS HOSTS ADDRESS PORTS AGE
colors-ingress <none> * 80 16s
```

**Task 4 - Access Pods From a single external IP**

Open the browser and enter the exnteral IP of the Ingress controller service and append "/blue/" to the path: http://"externalip"/blue/ (make sure to include the last /).

Open the browser and enter the exnteral IP of the Ingress controller service and append "/red/" to the path: http://"externalip"/red/ (make sure to include the last /)


### Shutdown or Delete the Cluster (cloud provider only)

When you're done for the day, you can shutdown your cluster to avoid incurring charges when you're not using it. This will delete all yournodes, but keep your configuration intact.

``` PS
az aks stop --name $AKS_NAME ` --resource-group $AKS_RESOURCE_GROUP
```

Or you can choose to delete the entier resource group instead. You can create a new one prior to the next lab.

``` PS
az group delete --resource-group $AKS_RESOURCE_GROUP
```
