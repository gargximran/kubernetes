

# Deploy and configure workload identity on an Azure Kubernetes Service (AKS) cluster

Azure Kubernetes Service (AKS) is a managed Kubernetes service that lets you quickly deploy and manage Kubernetes clusters. In this article, you will:

* Deploy an AKS cluster using the Azure CLI that includes the OpenID Connect Issuer and a Microsoft Entra Workload ID
* Grant access to your Azure Key Vault
* Create a Microsoft Entra Workload ID and Kubernetes service account
* Configure the managed identity for token federation.

## Prerequisite

- Azure CLI version 2.47.0 or later.

- The identity you're using to create your cluster has the appropriate minimum permissions.

- If you have multiple Azure subscriptions, select the appropriate subscription ID in which the resources should be billed using the ```az account``` command



## Export environment variables

To help simplify steps to configure the identities required, the steps below define
environment variables for reference on the cluster.

Run the following commands to create these variables. Replace the default values for `RESOURCE_GROUP`, `LOCATION`, `SERVICE_ACCOUNT_NAME`, `SUBSCRIPTION`, `USER_ASSIGNED_IDENTITY_NAME`, and `FEDERATED_IDENTITY_CREDENTIAL_NAME`.

```bash
export RESOURCE_GROUP="myResourceGroup"
export LOCATION="westcentralus"
export SERVICE_ACCOUNT_NAMESPACE="default"
export SERVICE_ACCOUNT_NAME="workload-identity-sa"
export SUBSCRIPTION="$(az account show --query id --output tsv)"
export USER_ASSIGNED_IDENTITY_NAME="myIdentity"
export FEDERATED_IDENTITY_CREDENTIAL_NAME="myFedIdentity"
```

## Create AKS cluster

Create an AKS cluster using the ```az aks create``` command with the `--enable-oidc-issuer` parameter to use the OIDC Issuer. The following example creates a cluster named *myAKSCluster* with one node in the *myResourceGroup*:

```azurecli-interactive
az aks create -g "${RESOURCE_GROUP}" -n myAKSCluster --enable-oidc-issuer --enable-workload-identity --generate-ssh-keys
```

After a few minutes, the command completes and returns JSON-formatted information about the cluster.



## Update an existing AKS cluster

You can update an AKS cluster using the ```az aks update``` command with the `--enable-oidc-issuer` and the `--enable-workload-identity` parameter to use the OIDC Issuer and enable workload identity. The following example updates a cluster named *myAKSCluster*:

```azurecli-interactive
az aks update -g "${RESOURCE_GROUP}" -n myAKSCluster --enable-oidc-issuer --enable-workload-identity
```

## Retrieve the OIDC Issuer URL

To get the OIDC Issuer URL and save it to an environmental variable, run the following command. Replace the default value for the arguments `-n`, which is the name of the cluster:

```bash
export AKS_OIDC_ISSUER="$(az aks show -n myAKSCluster -g "${RESOURCE_GROUP}" --query "oidcIssuerProfile.issuerUrl" -o tsv)"
```

The variable should contain the Issuer URL similar to the following example:

```output
https://eastus.oic.prod-aks.azure.com/00000000-0000-0000-0000-000000000000/11111111-1111-1111-1111-111111111111/
```

By default, the Issuer is set to use the base URL `https://{region}.oic.prod-aks.azure.com/{tenant_id}/{uuid}`, where the value for `{region}` matches the location the AKS cluster is deployed in. The value `{uuid}` represents the OIDC key.

## Create a managed identity

Use the Azure CLI ```az account set``` command to set a specific subscription to be the current active subscription. Then use the ```az identity create``` command to create a managed identity.

```azurecli-interactive
az identity create --name "${USER_ASSIGNED_IDENTITY_NAME}" --resource-group "${RESOURCE_GROUP}" --location "${LOCATION}" --subscription "${SUBSCRIPTION}"
```

Next, let's create a variable for the managed identity ID.

```bash
export USER_ASSIGNED_CLIENT_ID="$(az identity show --resource-group "${RESOURCE_GROUP}" --name "${USER_ASSIGNED_IDENTITY_NAME}" --query 'clientId' -o tsv)"
```

## Create Kubernetes service account

Create a Kubernetes service account and annotate it with the client ID of the managed identity created in the previous step. Use the ```az aks get-credentials``` command and replace the values for the cluster name and the resource group name.

```azurecli-interactive
az aks get-credentials -n myAKSCluster -g "${RESOURCE_GROUP}"
```

Copy and paste the following multi-line input in the Azure CLI.

```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: ServiceAccount
metadata:
  annotations:
    azure.workload.identity/client-id: "${USER_ASSIGNED_CLIENT_ID}"
  name: "${SERVICE_ACCOUNT_NAME}"
  namespace: "${SERVICE_ACCOUNT_NAMESPACE}"
EOF
```

The following output resembles successful creation of the identity:

```output
serviceaccount/workload-identity-sa created
```

## Establish federated identity credential

Use the ```az identity federated-credential create``` command to create the federated identity credential between the managed identity, the service account issuer, and the subject.

```azurecli-interactive
az identity federated-credential create --name ${FEDERATED_IDENTITY_CREDENTIAL_NAME} --identity-name "${USER_ASSIGNED_IDENTITY_NAME}" --resource-group "${RESOURCE_GROUP}" --issuer "${AKS_OIDC_ISSUER}" --subject system:serviceaccount:"${SERVICE_ACCOUNT_NAMESPACE}":"${SERVICE_ACCOUNT_NAME}" --audience api://AzureADTokenExchange
```

> [!NOTE]
> It takes a few seconds for the federated identity credential to be propagated after being initially added. If a token request is made immediately after adding the federated identity credential, it might lead to failure for a couple of minutes as the cache is populated in the directory with old data. To avoid this issue, you can add a slight delay after adding the federated identity credential.

## Deploy your application

When you deploy your application pods, the manifest should reference the service account created in the **Create Kubernetes service account** step. The following manifest shows how to reference the account, specifically *metadata\namespace* and *spec\serviceAccountName* properties:

```yml
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: your-pod
  namespace: "${SERVICE_ACCOUNT_NAMESPACE}"
  labels:
    azure.workload.identity/use: "true"  # Required, only the pods with this label can use workload identity
spec:
  serviceAccountName: "${SERVICE_ACCOUNT_NAME}"
  containers:
    - image: <your image>
      name: <containerName>
EOF
```

> [!IMPORTANT]
> Ensure your application pods using workload identity have added the following label `azure.workload.identity/use: "true"` to your pod spec, otherwise the pods fail after their restarted.



## Disable workload identity

To disable the Microsoft Entra Workload ID on the AKS cluster where it's been enabled and configured, you can run the following command:

```azurecli-interactive
az aks update --resource-group "${RESOURCE_GROUP}" --name myAKSCluster --disable-workload-identity
```
