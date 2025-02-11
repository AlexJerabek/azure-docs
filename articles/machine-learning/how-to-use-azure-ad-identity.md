---
title: Use Azure AD identity with your web service
titleSuffix: Azure Machine Learning
description: Use Azure AD identity with your web service in Azure Kubernetes Service to access cloud resources during scoring.
services: machine-learning
ms.author: larryfr
author: BlackMist
ms.reviewer: aashishb
ms.service: machine-learning
ms.subservice: enterprise-readiness
ms.date: 10/21/2021
ms.topic: how-to

---

# Use Azure AD identity with your machine learning web service in Azure Kubernetes Service

In this how-to, you learn how to assign an Azure Active Directory (Azure AD) identity to your deployed machine learning model in Azure Kubernetes Service. The [Azure AD Pod Identity](https://github.com/Azure/aad-pod-identity) project allows applications to access cloud resources securely with Azure AD by using a [Managed Identity](../active-directory/managed-identities-azure-resources/overview.md) and Kubernetes primitives. This allows your web service to securely access your Azure resources without having to embed credentials or manage tokens directly inside your `score.py` script. This article explains the steps to create and install an Azure Identity in your Azure Kubernetes Service cluster and assign the identity to your deployed web service.

## Prerequisites

- The [Azure CLI extension for the Machine Learning service](reference-azure-machine-learning-cli.md), the [Azure Machine Learning SDK for Python](/python/api/overview/azure/ml/intro), or the [Azure Machine Learning Visual Studio Code extension](how-to-setup-vs-code.md).

- Access to your AKS cluster using the `kubectl` command. For more information, see [Connect to the cluster](../aks/learn/quick-kubernetes-deploy-cli.md#connect-to-the-cluster)

- An Azure Machine Learning web service deployed to your AKS cluster.

## Create and install an Azure Identity

1. To determine if your AKS cluster is Kubernetes RBAC enabled, use the following command:

    ```azurecli-interactive
    az aks show --name <AKS cluster name> --resource-group <resource group name> --subscription <subscription id> --query enableRbac
    ```

    This command returns a value of `true` if Kubernetes RBAC is enabled. This value determines the command to use in the next step.

1. Install [Azure AD Pod Identity](https://azure.github.io/aad-pod-identity/docs/getting-started/installation/) in your AKS cluster.

1. [Create an Identity on Azure](https://azure.github.io/aad-pod-identity/docs/demo/standard_walkthrough/#2-create-an-identity-on-azure) following the steps shown in Azure AD Pod Identity project page.

1. [Deploy AzureIdentity](https://azure.github.io/aad-pod-identity/docs/demo/standard_walkthrough/#3-deploy-azureidentity) following the steps shown in Azure AD Pod Identity project page.

1. [Deploy AzureIdentityBinding](https://azure.github.io/aad-pod-identity/docs/demo/standard_walkthrough/#5-deploy-azureidentitybinding) following the steps shown in Azure AD Pod Identity project page.

1. If the Azure Identity created in the previous step is not in the same node resource group for your AKS cluster, follow the [Role Assignment](https://azure.github.io/aad-pod-identity/docs/getting-started/role-assignment/#user-assigned-identities-that-are-not-within-the-node-resource-group) steps shown in Azure AD Pod Identity project page.

## Assign Azure Identity to web service

The following steps use the Azure Identity created in the previous section, and assign it to your AKS web service through a **selector label**.

First, identify the name and namespace of your deployment in your AKS cluster that you want to assign the Azure Identity. You can get this information by running the following command. The namespaces should be your Azure Machine Learning workspace name and your deployment name should be your endpoint name as shown in the portal.

```azurecli-interactive
kubectl get deployment --selector=isazuremlapp=true --all-namespaces --show-labels
```

Add the Azure Identity selector label to your deployment by editing the deployment spec. The selector value should be the one that you defined in step 5 of [Deploy AzureIdentityBinding](https://azure.github.io/aad-pod-identity/docs/demo/standard_walkthrough/#5-deploy-azureidentitybinding).

```yaml
apiVersion: "aadpodidentity.k8s.io/v1"
kind: AzureIdentityBinding
metadata:
  name: demo1-azure-identity-binding
spec:
  AzureIdentity: <a-idname>
  Selector: <label value to match>
```

Edit the deployment to add the Azure Identity selector label. Go to the following section under `/spec/template/metadata/labels`. You should see values such as `isazuremlapp: “true”`. Add the aad-pod-identity label like shown below.

```azurecli-interactive
    kubectl edit deployment/<name of deployment> -n azureml-<name of workspace>
```

```yaml
spec:
  template:
    metadata:
      labels:
       aadpodidbinding: "<value of Selector in AzureIdentityBinding>"
      ...
```

To verify that the label was correctly added, run the following command. You should also see the statuses of the newly created pods.

```azurecli-interactive
   kubectl get pod -n azureml-<name of workspace> --show-labels
```

Once the pods are up and running, the web services for this deployment will now be able to access Azure resources through your Azure Identity without having to embed the credentials in your code.

## Assign roles to your Azure Identity

[Assign your Azure Managed Identity with appropriate roles](../active-directory/managed-identities-azure-resources/how-to-manage-ua-identity-portal.md) to access other Azure resources. Ensure that the roles you are assigning have the correct **Data Actions**. For example, the [Storage Blob Data Reader Role](../role-based-access-control/built-in-roles.md#storage-blob-data-reader) will have read permissions to your Storage Blob while the generic [Reader Role](../role-based-access-control/built-in-roles.md#reader) might not.

## Use Azure Identity with your web service

Deploy a model to your AKS cluster. The `score.py` script can contain operations pointing to the Azure resources that your Azure Identity has access to. Ensure that you have installed your required client library dependencies for the resource that you are trying to access to. Below are a couple examples of how you can use your Azure Identity to access different Azure resources from your service.

### Access Key Vault from your web service

If you have given your Azure Identity read access to a secret inside a **Key Vault**, your `score.py` can access it using the following code.

```python
from azure.identity import DefaultAzureCredential
from azure.keyvault.secrets import SecretClient

my_vault_name = "yourkeyvaultname"
my_vault_url = "https://{}.vault.azure.net/".format(my_vault_name)
my_secret_name = "sample-secret"

# This will use your Azure Managed Identity
credential = DefaultAzureCredential()
secret_client = SecretClient(
    vault_url=my_vault_url,
    credential=credential)
secret = secret_client.get_secret(my_secret_name)
```

> [!IMPORTANT]
> This example uses the DefaultAzureCredential. To grant your identity access using a specific access policy, see [Assign a Key Vault access policy using the Azure CLI](../key-vault/general/assign-access-policy-cli.md).

### Access Blob from your web service

If you have given your Azure Identity read access to data inside a **Storage Blob**, your `score.py` can access it using the following code.

```python
from azure.identity import DefaultAzureCredential
from azure.storage.blob import BlobServiceClient

my_storage_account_name = "yourstorageaccountname"
my_storage_account_url = "https://{}.blob.core.windows.net/".format(my_storage_account_name)

# This will use your Azure Managed Identity
credential = DefaultAzureCredential()
blob_service_client = BlobServiceClient(
    account_url=my_storage_account_url,
    credential=credential
)
blob_client = blob_service_client.get_blob_client(container="some-container", blob="some_text.txt")
blob_data = blob_client.download_blob()
blob_data.readall()
```

## Next steps

* For more information on how to use the Python Azure Identity client library, see the [repository](https://github.com/Azure/azure-sdk-for-python/tree/master/sdk/identity/azure-identity#azure-identity-client-library-for-python) on GitHub.
* For a detailed guide on deploying models to Azure Kubernetes Service clusters, see the [how-to](how-to-deploy-azure-kubernetes-service.md).