
# Lab - Advanced Kubernetes Topics

Estimated Duration: 60 minutes

Exercises
- Create a Basic AKS Cluster
-Exercise: Using Secrets Stored in Azure Key Vault
- Exercise: Using Taints, Tolerations, Node Selectors and Affinity
- Shutdown or Delete the AKS Cluster

### Create a Basic AKS Cluster

For the exercises in this module, you'll need simple AKS cluster.

**Task 1 - Create an AKS cluster (or start an existing one)**
	
Select the region closest to your location. Use 'eastus' for United States workshops, 'westeurope' for European workshops. Askyour instructor for other options in your region: `eastus`, `westus`, `canadacentral`, `westeurope`, `centralindia`, `australiaeast`

Define variables (update as needed)

``` PS
$INITIALS="abc"
$YOUR_INITIALS="$($INITIALS)".ToLower()
$AKS_RESOURCE_GROUP="azure-$($INITIALS)-rg"
$VM_SKU="Standard_D2as_v5"
$AKS_NAME="aks-$($INITIALS)"
$NODE_COUNT="3"
$LOCATION="your region"
```

Create Resource Group

``` PS
az group create --location $LOCATION ` --resource-group $AKS_RESOURCE_GROUP
```

Create Basic cluster.

``` PS
az aks create --node-count $NODE_COUNT `
    --generate-ssh-keys `
    --node-vm-size $VM_SKU `
    --name $AKS_NAME `
    --resource-group $AKS_RESOURCE_GROUP
```

Connect to local environment

``` PS
az aks get-credentials --name $AKS_NAME ` --resource-group $AKS_RESOURCE_GROUP
```

Verify connection

``` PS
kubectl get nodes
```

### Exercise: Using Secrets Stored in Azure Key Vault

In this Azure Key Vault exercise, you'll perform the following actions:
- Enable Key Vault Addon in AKS - Azure installs all the components need to integrate Key Vault with AKS
- Create an Azure Key Vault - This will contain all the secrets you'll use
- Grant administrator permissions you your account - This will allow you to create secrets in the Key Vault.
- Create a secret in Azure Key Vault - This will represent the sensitive data you should keep outside the cluster.
- Grant reader permissions to the AKS cluster - This will allow the cluster to read the external secret
- Create custom resources in your cluster to establish the connection to Key Vault - This will specify the AKS identity, Key Vaultand secret that will be injected into your Pod.
- Mount a CSI secrets volume in your Pod - This will use the information in custom resource retrieve and mount your secret.

**Task 1 - Enable Key Vault Addon in AKS and create a Key Vault**

Define variables.

``` PS
$INITIALS="abc"
$YOUR_INITIALS="$($INITIALS)".ToLower()
$AKS_RESOURCE_GROUP="azure-$($INITIALS)-rg"
$LOCATION="your region"
$AKS_IDENTITY="identity-$($INITIALS)"
$AKS_NAME="aks-$($INITIALS)"
$KV_NAME="kv-$($INITIALS)"
```

Enable Key Vault Addon in AKS

``` PS
az aks addon enable `
    --addon azure-keyvault-secrets-provider `
    --name $AKS_NAME `
    --resource-group $AKS_RESOURCE_GROUP
```

Verify addon has been enabled

``` PS
az aks addon list --name $AKS_NAME --resource-group $AKS_RESOURCE_GROUP -o table
```

Create Key Vault. Notice the `--enable-rbac-authorization` option. This will allow AKS to use its managed identity to access theKey Vault.

``` PS
az keyvault create --name $KV_NAME ` --resource-group $AKS_RESOURCE_GROUP ` --enable-rbac-authorization
```

**Task 2 - Assign Permissions for you to create Secrets from Key Vault**

List the secrets in the Key vault.

``` PS
az keyvault secret list --vault-name $KV_NAME
```

Even though you created the Key Vault, you don't automatically have access to the Data Plane of the vault. You must give yourselfadditional permissions first.

You can assign yourself as the Key Vault Secrets Officer, which will allow you to maintain secrets. You may wish to use Key VaultAdministrator instead, which will allow you create Keys and Certificates as well.

Get the object id of the Key Vault:

``` PS
$KV_ID=(az keyvault list --query "[? name=='$($KV_NAME)'].{id:id}" -o tsv)
```

Get your object id:

``` PS
$CURRENT_USER_ID=(az ad signed-in-user show --query "{objectId:objectId}" -o tsv)
```

Assign yourself the Key Vault Secrets Officer/Administrator role:

``` PS
az role assignment create `
    --role "Key Vault Secrets Officer" `
    --assignee-object-id $CURRENT_USER_ID `
    --scope $KV_ID
```

List the secrets in the Key Vault again. This time you should get an empty list instead of "not authorized" error.

``` PS
az keyvault secret list --vault-name $KV_NAME
```

Create a sample secret for testing

``` PS
az keyvault secret set ` --vault-name $KV_NAME ` --name SampleSecret ` --value "Highly sensitive information: <place data here>"
```

Verify the secret is in the Key Vault

``` PS
az keyvault secret list --vault-name $KV_NAME -o table
```

You should now see the secret listed.

**Task 3 - Assign Permissions to AKS to read Secrets from Key Vault**

You'll need to assign AKS the Key Vault Secrets User role so it's able to read the secrets you created.

List all available identities. Find the managed identity created for accessing the Key Vault from AKS when the addon was enabled.

``` PS
az identity list --query "[].{name:name,ClientId:clientId}" -o table
```

Get the object id of that managed identity AKS is using

``` PS
$KV_IDENTITY=(az identity list --query "[? contains(name,'azurekeyvaultsecretsprovider')].{principalId:principalId}" -o tsv)
$KV_CLIENT_ID=(az identity list --query "[? contains(name,'azurekeyvaultsecretsprovider')].{clientId:clientId}" -o tsv)
```

Assign the AKS identity permission to read secrets from the Key Vault.

``` PS
az role assignment create ` --role "Key Vault Secrets User" ` --assignee-object-id $KV_IDENTITY ` --scope $KV_ID
```

**Task 4 - Create Kubernetes resources and read secret from Key Vault**

Gather the needed values

``` PS
$TENANT_ID=(az aks show --name $AKS_NAME --resource-group $AKS_RESOURCE_GROUP --query "{tenantId:identity.tenantId}" -o tsv)
write-host @"Client Id: $($KV_CLIENT_ID)Key Vault Name: $($KV_NAME)Tenant Id: $($TENANT_ID)"@
```

Change current folder to Module6

``` PS
cd 03-advanced
```

Open the file called called spc.yaml.

``` PS
code spc.yaml
```

Replace the placeholders with the values listed above.

``` yaml
apiVersion: secrets-store.csi.x-k8s.io/v1
kind: SecretProviderClass
metadata: 
  name: azure-kv-secret
spec: 
  provider: azure 
  parameters: 
    usePodIdentity: "false" 
    useVMManagedIdentity: "true" 
    userAssignedIdentityID: <client-id> 
    keyvaultName: <key-vault-name> 
    cloudName: "" 
    objects: | 
      array: 
      - | 
        objectName: SampleSecret 
        objectType: secret 
        objectVersion: "" 
        tenantId: <tenant-id>
```

Apply the manifest.

``` PS
kubectl apply -f spc.yaml
```

Review the contents of pod-kv.yaml.

``` yaml
kind: Pod
apiVersion: v1
metadata: 
  name: pod-kv
spec: 
  containers:
  - name: busybox 
    image: k8s.gcr.io/e2e-test-images/busybox:1.29-1 
    command:
    - "/bin/sleep"
    - "10000"
    volumeMounts:
    - name: secrets-store01 
      mountPath: "/mnt/secrets-store" 
      readOnly: true 
      volumes:
      - name: secrets-store01
        csi: 
          driver: secrets-store.csi.k8s.io 
          readOnly: true 
          volumeAttributes: 
            secretProviderClass: "azure-kv-secret"
```

Apply the manifest.

``` PS
kubectl apply -f pod-kv.yaml
```

View secret value in Pod

``` PS
kubectl exec -it pod-kv -- cat /mnt/secrets-store/SampleSecret
```

You should now be able to see the content of the Key Vault secret you created earlier.

**Task 5 - Create a Kubernetes Secret from a secret in Key Vault**

Many Kubernetes resources use Secret resources (Ingress, Persistent Volumes, etc.). You can extend the configuration about to create aKubernetes secret object when you mount the Pod.

Edit the SecretProviderClass you created earlier.

``` PS
code spc.yaml
```

Add the indicated section.

``` yaml
apiVersion: secrets-store.csi.x-k8s.io/v1
kind: SecretProviderClass
metadata: 
  name: azure-kv-secret
spec: 
  provider: azure 
  ########## Add this section ######### 
  secretObjects:
  - data:
    - key: MySecret 
      objectName: SampleSecret 
      secretName: k8s-secret 
      type: Opaque 
  ########## End of section #########
  parameters: 
    usePodIdentity: "false" 
    useVMManagedIdentity: "true" 
...
```

Update the object.

``` PS
kubectl apply -f spc.yaml
```

Edit the Pod manifest. Notice the env section setting the environment variable.

``` PS
code pod-kv.yaml
```

``` yaml
kind: Pod
apiVersion: v1
metadata: 
  name: pod-kv
spec: 
  containers:
  - name: busybox 
    image: k8s.gcr.io/e2e-test-images/busybox:1.29-1 
    command:
    - "/bin/sleep"
    - "10000" 
    volumeMounts: 
    - name: secrets-store01 
      mountPath: "/mnt/secrets-store" 
      readOnly: true 
    ########## Add this section ######### 
    env:
    - name: SECRET_VALUE 
      valueFrom: 
        secretKeyRef: 
          name: k8s-secret 
          key: MySecret 
    ########## End of section ######### 
  volumes:
  - name: secrets-store01
    csi: 
      driver: secrets-store.csi.k8s.io 
      readOnly: true 
      volumeAttributes: 
        secretProviderClass: "azure-kv-secret"
```

Since Pods are immutable, you'll have to delete it and then reapply the manifest.

``` PS
kubectl delete pod pod-kvkubectl apply -f pod-kv.yaml
```

View the secret in the Pod.

``` PS
kubectl exec -it pod-kv -- printenv
```

Verify the Secret object has been created and is available for other Pods to use.

``` PS
kubectl get secret
```

Delete the Pod and the Kubernetes Secret object will also be deleted. Once an injected Secret is no longer referenced by anyPods, it's automatically deleted.

``` PS
kubectl delete pod pod-kvkubectl get secret
```

Once an injected Secret is no longer referenced by any Pods, it's automatically deleted.

### Exercise: Using Taints, Tolerations, Node Selectors and Affinity

In this exercise, you will work with 3 simple workloads of 12 Pods each. 
You'll ensure that "heavy" workloads are scheduled on Nodes thatcan handle the loads and "light" workloads don't clutter up those nodes and use up valuable resources which could be used by the Podsthat need them.

**Scenario**

We assume you have 3 nodes in your cluster. If you don't, please create and AKS cluster with at least 3 nodes.
Assume the following about your AKS cluster:
- One of the nodes has specialized hardware (GPUs) which make it ideal for hosting graphics processing workloads and forMachine Learning.
- The other 2 nodes have basic hardware and are meant for general purpose processing.
- You have 3 workloads you need to schedule in the cluster:

`Workload 1` – Makes use of heavy GPU hardware for processing tasks.
`Workload 2` – General purpose
`Workload 3` – General purpose


The goal of this exercise is to make sure that:
- Workload 1 Pods are scheduled ONLY on Node 1, so they can take full advantage of the hardware.
- Workloads 2 & 3 are NOT scheduled on Node 1, so they don't use up resources which could be used by additional replicas ofWorkload 1.

You'll start with 3 workload yaml files. The containers in them are identical, but for the purpose of this lab, you'll pretend andworkload1.yaml does extensive processing that takes advantage of a GPU in the Node.

**Task 1 – Add workloads to the cluster.**

Open a Window Terminal window and apply the following workloads:

``` PS
kubectl apply -f .\workload-1.yamlkubectl apply -f .\workload-2.yamlkubectl apply -f .\workload-3.yaml
```

Verify that the Pods are running

``` PS
kubectl get pods
```

**Task 2 – Add color and label to node.**

Get a list of the Nodes.

``` PS
kubectl get nodes
```

Pick on of the Nodes to be the "GPU" Node (it doesn't matter which one). Copy it's name by clicking the Right button on your mouse

Set a variable to the Node name you selected (replace the NodeName with your selected node)

``` PS
$NodeName="aks-userpool1-20365992-vmss000001"
```

Add a color=lime and process=GPU labels to the Node to distinguish it from the others and allow the Pods to find it..

``` PS
kubectl label node $NodeName color=lime, process=GPU --overwrite
```

**Task 3 – Update the Node Selector of the Workload.**

Edit the workload-1.yaml file to update the node selector to ensure it's only scheduled on the selected node. 

``` yaml
nodeSelector: 
  kubernetes.io/os: linux 
  process: GPU
```

Save and apply workload1.yaml again.

``` PS
kubectl apply -f .\workload-1.yaml
```

Verify that the Pods have been rescheduled on the selected node.

``` PS
kubectl get pods -o wide
```

Notice also that some of other pods are on the same node. If you scale workload-1, there might not be enough room for the additional pods to be scheduled on the selected node. Those pods will remain in a Pending state because any label specified inthe NodeSelector is a required to be present on the node before a Pod can be scheduled there. Since no other nodes have the required label, there's nowhere for the scheduler to place the additional Pods.

**Task 4 – Add a Taint to the Node to prevent other Pods from being scheduled on it.**

Add a NoSchedule taint to the selected node

``` PS
kubectl taint nodes $NodeName allowed=GPUOnly:NoSchedule
```

The problem is that there are still pods on this node that were scheduled before the node was tainted. Remove those pods andlet the scheduler place the on different nodes

``` PS
kubectl delete pods --field-selector=spec.nodeName=$NodeName
```

Get a list of Pods

``` PS
kubectl get pods
```

As you can see the other Pods were rescheduled. However, ** workload-1** pods are all in a Pending state.

Describe one of the Pods to see the problem (substritute one of your pod names)

``` PS
kubectl describe pod workload-1-7d7499b8f4-4jmck
```

**Task 5 – Add a Toleration to the Pods so they can be scheduled on the selected Node.**

Add a toleration to the workload-1.yaml file (same indentation and NodeSelector) 

``` yaml
tolerations:
- key: "allowed" 
  operator: "Equal" 
  value: "GPUOnly" 
  effect: "NoSchedule"
```

Save and apply the changes.

``` PS
kubectl apply -f .\workload-1.yaml
```

Review the pods

``` PS
kubectl get pods -o wide
```

Now you can see that all the Workload-1 pods are on the selected node AND no other pods are scheduled (or will bescheduled) on the selected node.

### Shutdown or Delete the AKS Cluster

When you're done for the day, you can shutdown your cluster to avoid incurring charges when you're not using it. This will delete all yournodes, but keep your configuration intact.

``` PS
az aks stop --name $AKS_NAME ` --resource-group $AKS_RESOURCE_GROUP
```

Or you can choose to delete the entire resource group instead. You can create a new one prior to the next lab.

``` PS
az group delete --resource-group $AKS_RESOURCE_GROUP
```
