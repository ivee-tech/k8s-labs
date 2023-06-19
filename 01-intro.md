# Lab - Core Kubernetes Concepts

Estimated Duration: 60 minutes

Exercices

- Exercise: Verify Kubernetes Cluster
- Exercise: Creating a Pod Declaratively
- Exercise: Adding/Updating/Deleting Labels on a Pod
- Exercise: Working with Deployments
- Exercise: Working with Services
- Exercise: Cleanup

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

### Exercise: Creating a Pod Declaratively

This Exercise demonstrates the use of a YAML file to create a pod declaratively.

**Task 1 - Create a Pod declaratively**

Change into the **01-intro** folder

``` PS
cd 01-intro
```

Use the YAML file provided to create a Pod. You may want to open the simple-pod.yaml file and review its contents.

The pod definition contains the Nginx container that listens to port 80.

``` PS
kubectl apply -f simple-pod.yaml
```

Now, make sure pod is up and running.

``` PS
kubectl get pods
```

You should see a pod named nginx-pod
Add a second pod, then check the list again.

``` PS
kubectl apply -f simple-pod2.yaml
kubectl get pods
```

**Task 2 - Filter pods based on a label**

Show all the labels in the pods

``` PS
kubectl get pods --show-labels
```
	
Let's say you want to list pods that have a label named kind=web associated with them. You can use -l switch to apply filterbased on labels.

``` PS
kubectl get pod -l kind=web
```

To prove that this works as expected, run the command again but change the value of label kind to db. Notice, this time kubectl doesn't return any pods because there are no pods that match the label kind and a value of db.

``` PS
kubectl get pod -l kind=db
```

**Task 3 - View complete definition of the Pod**

Query Kubernetes to return the complete definition of a Pod from its internal database by exporting the output (-o) to YAML.
Then pipe the result to a file.

``` PS
kubectl get pods nginx-pod -o yaml > mypod.yaml
```

To view the JSON version, use the -o json flag instead.
View the contents of the generated file in VS Code (or an editor of your choice).

``` PS
code mypod.yaml
```

> **_NOTE:_** Observe all the properties that Kubernetes populated with default values when it saved the Pod definition to its database.

### Exercise: Adding/Updating/Deleting Labels on a Pod

In this Exercise, you will create a pod that has labels associated with it. Labels make it easy to filter the pods later. Labels play a vital rolein the Kubernetes ecosystem, so it's important to understand their proper usage.

**Task 1 - Assign a new label to a running Pod**

Assign a new label (key=value) pair to a running pod. This comes in handy when you are troubleshooting an issue and wouldlike to distinguish between different pod(s). Assign a new label health=fair to the pod nginx-pod, which is already running.

``` PS
kubectl label pod nginx-pod health=fair
```

Run the command below to show the pod labels. Notice that now an additional label is shown with the pod.

``` PS
kubectl get pods nginx-pod --show-labels
```

**Task 2 - Update an existing label that is assigned to a running pod**

Update the value of an existing label that is assigned to a running pod. Change the value of the label kind=web to kind=db ofthe nginx-pod pod.

``` PS
kubectl label pod nginx-pod kind=db --overwrite
```

`--overwrite` is needed because the pod is running and won't accept changes otherwise.

Show the pod labels again. Notice that kind has changed from web to db.

``` PS
kubectl get pods --show-labels
```

**Task 3 - Delete a label that is assigned to a running Pod**

Delete the label health from the nginx-pod pod.

``` PS
kubectl label pod nginx-pod health-
```

> **_NOTE:_** Notice the minus (-) sign at the end of the command. You can also remove a label from all running pods by using the --all flag.

``` PS
kubectl label pod health- --all
```

Run the command below to show the pod labels again. Notice that health is not part of the list of labels.

``` PS
kubectl get pods --show-labels
```

**Task 4 - Delete Pods based on their labels**

Delete all the Pods that match a specific label.

``` PS
kubectl delete pod -l target=dev
```

### Exercise: Working with Deployments

In this Exercise, you will create a Deployment and rollout an application update. Deployments provide a consistent mechanism to upgrade an application to a new version, while keeping the downtime to a minimum. 

Note that internally, Deployments use ReplicaSets for managing Pods. However, you never work directly with ReplicaSets since Deployments abstract out that interaction.

**Task 1 - Create a new Deployment**

The ng-dep.yaml file contains a Deployment manifest. The Pod in the template contains an nginx container with a tag 1.0. The 1.0 represents the version of this container and hence of the application running inside it.

Create a Deployment and a Service to access the Pods of the deployment.

``` PS
kubectl apply -f ng-dep.yamlkubectl apply -f ng-svc.yaml
```

> **_NOTE:_** The --record flag saves the command you applied in the deployment's ReplicaSet history. This helps in deciding which previous Revision to roll back to if needed.

Run the following command to see the Pods, ReplicaSets, Deployments and Services that were created.

``` PS
kubectl get all --show-labels
```

**Task 2 - Access version 1.0 of application (only for cloud provider)**

Wait about 3-4 minutes to allow Azure to create a Public IP address for the service. Check to see if an address has been assignedby getting the list of services.

``` PS
kubectl get svc
```

When you see an EXTERNAL-IP assigned, open a browser with that address. Example: http://20.81.24.216

**Task 3 - Update the Deployment to version 2.0**

You are now going to update the Deployment to use version 2.0 of the container instead of 1.0. This can be done in one of two ways.One approach is to use imperative syntax, which is faster and is often used during the development/testing stage of an application. The alternate method is to update the YAML file and then to reapply it to the cluster.

To start rolling out the new update, change the container image tag from 1.0 to 2.0 by running this command:

``` PS
kubectl set image deployment ng-dep nginx=k8slab/nginx:2.0
```

In the command above, ng-dep is the name of Deployment and nginx is the name of the container within the Pod template.The change will force the Deployment to create a new ReplicaSet with an image tagged 2.0.

List all the pods and notice that old pods are terminating and that new Pods have been created.

``` PS
kubectl get pods
```

Run the follwing command to review the Deployment definition with the updated value of container image:

``` PS
kubectl describe deployment ng-dep
```

Notice the Image section (under Containers) shows the value of container image as 2.0.

Run the command to view the Pods, ReplicaSets and Deployments again.

``` PS
kubectl get all
```

Notice that the old replica set still exists, even though it has 0 Desired Pods.

Run the describe command on that old ReplicaSet.

``` PS
kubectl describe rs <old replicaset name>
```

Notice that the old definition still has the previous version number. This is maintained so you can roll back the change to that version ifyou which.

Access the 2.0 version of application by refreshing the browser at the same address as above.

**Task 4 - Rollback the Deployment**

The purpose of maintaining the previous ReplicaSet is to be able to rollback changes to any previous version.
Review the deployment history.

``` PS
kubectl rollout history deploy/ng-dep
```

Rollback the Deployment to the previous version.

``` PS
kubectl rollout undo deploy/ng-dep
```

Wait a few seconds and refresh the browser again.

Notice the site is back to the previous version.

**Task 5 - Delete the Deployment and Service**

Delete the Deployment and Service.

``` PS
kubectl delete deployment ng-depkubectl delete service ng-svc
```

> **_NOTE:_** It may take a few minutes to delete the service because has to delete the Public IP resource in Azure.

### Exercise: Working with Services

In this Exercise you will create a simple Service. Services help you expose Pods externally using label selectors.

**Task 1 - Create a new Service**

Create a deployment.

``` PS
kubectl apply -f sample-dep.yaml
```

The sample-svc.yaml file contains a Service manifest. Services use label selectors to determine which Pods it needs to track andforward the traffic to.

Review running Pods and their labels.

``` PS
kubectl get pods --show-labels
```

Notice the label sample=color that is associated with the Pods.
	
Open the sample-svc.yaml file and examine the selector attribute. Notice the sample: color selector. This Service will track all Pods that have a label sample=color and load balance traffic between them.

Create the Service.

``` PS
kubectl apply -f sample-svc.yaml
```

Check the of newly created service.

``` PS
kubectl get svc -o wide
```

The command above will display the details of all available services along with their label selectors. You should see the sample-svc Service with PORTS 80:30101/TCP and SELECTOR sample=color.

**Task 2 - Access the sample-svc Service**

Open a browser and navigate to the IP address shown in the output of the previous command.

The website displays the Node IP/Pod IP address of the pod currently receiving the traffic through the service's load balancer.The page refreshes every 3 seconds and each request may be directed to a different pod, with a different IP address. This is theservice's internal load balancer at work.

**Task 3 - Delete the Deployment and Service**

Deleting any Pod will simply tell Kubernetes that the Deployment is not in its desired state and it will create a replacement. You can onlydelete Pods by deleting the Deployment.

Delete the Deployment.

``` PS
kubectl delete deployment sample-dep
```

The Service is independent of the Pods it services, so it's not affected when the Deployment is deleted. Anyone trying to accessthe service's address will simply get a 404 error. If the Deployment is ever re-created, the Service will automatically start sendingtraffic to the new Pods.

Delete the Service.

``` PS
kubectl delete service sample-svc
```

### Exercise: Cleanup

**Task 1 - Delete the cluster (only for cloud provider)**

When you're done working with the cluster, you can delete it if you wish. This will ensure you don't incur any costs on your sponsorshipsubscription when you're not working on the labs.

Deleting the cluster is as easy as creating it (example for AKS).

``` PS
az aks delete --name $clusterName --resource-group $rg
```
