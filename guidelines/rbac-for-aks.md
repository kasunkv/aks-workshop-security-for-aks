# Creating AKS Cluster with Azure Active Directory Integration

Kubernetes does not have any objects that represent a user. Kubernetes expects users to be managed by an external service. To do the user management and to use RBAC in AKS we can integrate the AKS cluster with Azure Active Directory. And the use RBAC to allow access to Kubernetes for individual Azure AD Users or Groups.

## Important To Know

* **Azure AD can only be integrated when creating the AKS Cluster.** This can not be done afterwards.
* **Guest Users in Azure AD are not Supported.** It has to be a user in the Azure AD itself.

# 1. Create the Azure AD Application Registrations

We will need 2 Azure AD Applications for this exercise.

* **AKS Azure AD Server** - This is to get access to Azure AD Users and Groups for granting access to AKS
* **AKS Azure AD Client** - This is used to login to Kubernetes with the Kubernetes CLI, `kubectl`

## A). Create AKS Azure AD Server Application Registration
---

## i. Create the Application Registration

* Go to Azure Portal and then **Azure Active Directory > App Registrations**
* Click **New Application Registration** button and enter the following details
    - _Name_ : `AKS Azure AD Server`
    - _Application type_ : `Web app / API`
    - _Sign-on URL_ : `http://aksazureadserver`

## ii. Give Access to Groups and Roles for the Logged-In User

* Click on **Manifest** button in the Application Blade
* In the **Edit Manifest** blade, change the `groupMembershipClaims`
    
    **Before**
    ```json
    {
        "groupMembershipClaims": null,
    }
    ```

    **After**
    ```json
    {
        "groupMembershipClaims": "All",
    }
    ```

## iii. Create the Application Secret Key & Copy the Application ID

* On the Application Blade, click on **Settings > Keys**
* Set the following values and click **Save**
    - _Description_ : `AppSecret`
    - _Expires_ : `Never Expires`
* Copy the **Value**. _This is a onetime operation, save it for later. You can't copy it again_
* Go to the Application Blade and copy the **Application ID**. Save it for later use.

## iv. Give Permission for the Application

* On the Application Blade, click on **Settings > Required Permissions > Add > Select an API**. Select `Microsoft Graph` and Click on **Select**
* On the _Select Permissions_,
    - Under **Application Permissions**
        - Select `Read directory data`
    - Under **Delegated Permissions**
        - Select `Sign in and read user profile`
        - Select `Read directory data`
* Click on **Select** and finally Click on **Done**
* Finally, click on `Microsoft Graph` list item and click `Grant Permissions` button.


## B). Create AKS Azure AD Client Application Registration - For `kubectl`
---

## i. Create the Application Registration

* Go to Azure Portal and then **Azure Active Directory > App Registrations**
* Click **New Application Registration** button and enter the following details
    - _Name_ : `AKS Azure AD Client`
    - _Application type_ : `Native`
    - _Sign-on URL_ : `http://aksazureadclient`

## ii. Give Permission for the Application

* On the Application Blade, click on **Settings > Required Permissions > Add > Select an API**. 
* Search for `AKS Azure AD Server` application and highlight it and then Click on **Select**
* Select the checkbox infront of `Access AKS Azure AD Server` option anc click on **Select** and the click on **Done**
* Finally, click on `AKS Azure AD Server` list item and click `Grant Permissions` button.


## iii. Copy the Application ID

* On the Application Blade, Copy `Application ID` and save it for later use.


## C). Copy the Tenant ID for the Azure AD
---

* On the Portal, go to **Azure Active Directory > Properties** 
* Then copy the **Directory ID**. This is your `Tenant ID`


## D). Create the Service Principle for AKS
---

We need to create a `Service Principal` to be used with AKS. We will use this Service Principle in a later exercise to give AKS access to Azure Container Registry.

```powershell
# Login to Azure Subscription
az login --use-device-code

# If you want to swtich subscriptions use the following command.
az acount set -s "<subscription-id>"

# Create the Service Pricipal
az ad sp create-for-rbac --name "sp-aks-security" --skip-assignment

# this will give a json object containing the appId and the password. Make note of these 2 values
# we need this for the next step
```


## E). Create the AKS Cluster with Azure AD Integration and Custom Service Principal
---

Let's use the Azure AD Applications and Service Principal we created to create an AKS cluster and associate it with Azure AD

```powershell
# Create the resource group for the AKS cluster
az group create --name "aks-security-rg" --location "southeastasia"

# Create the AKS cluster
az aks create `
    --resource-group "aks-security-rg" `
    --name "<aks-name>" `
    --generate-ssh-keys `
    --aad-server-app-id "<server-app-id>" `
    --aad-server-app-secret "<server-app-secret>" `
    --aad-client-app-id "<client-app-id>" `
    --aad-tenant-id "<tenant-id>" `
    --service-principal "<sp-app-id>" `
    --client-secret "<sp-password>"
```

## F). Create and Configure a User in the Azure Active Directory for Testing
---

We will be creating a user in the Azure AD and to do this we need to have a verified domain for the user. We will use the default domain for the Azure AD that can be fount in the Azure AD home blade.

### i). Create the User in the Azure AD

* In the Azure Portal, go to **Azure Active Directory**
* Copy the Default domain for the AAD from the Overview blade
* Then then click on **Users > New User** button
* Add the following details.
    - _Name_ : `John Doe`
    - _Username_ : `johnd@your-aad-domian.com`
    - _Add the other info if needed_
    - **Make sure to copy the password**
* Click **Create**


### ii). Assign Subscription Access to the User.
If you login with this newly created user, you will see that there is no subscription attached to this user.

* On the Portal goto **Subscription** and select your subscription.
* In the **Access Control (IAM)** blade assign `Contributor` Access for the subscription


### iii). Login with the Newly Created User and Change the Password

* Login to the Azure Portal with the new user and change the password
* _You will need to old password we copied for this_


## G). Create the Azure AD Groups
---

* In the Portal, goto **Azure Active Directory > Groups > New Group**
* Create 2 groups with the following values.
    - **Admin Group**
        - _Group Type_  : `Security`
        - _Group Name_  : `AKS-Admin`
        - _Description_ : optional
        - _Member Type_ : `Assigned`
    - **Reader Group**
        - _Group Type_  : `Security`
        - _Group Name_  : `AKS-Reader`
        - _Description_ : optional
        - _Member Type_ : `Assigned`


## H). Create `ClusterRoleBinding` to Give Access to AKS for the AAD Group
---

### i). Login to AKS with Admin Access

```powershell
# Login with Admin access
az aks get-credentials --resource-group "aks-security-rg" --name "<aks-name>" --admin
```

### ii). Bind the AKS-Group with `cluster-admin` Default `ClusterRole` in Kubernetes

Kubernetes already have a `ClusterRole` called _cluster-admin_. We can create a `ClusterRoleBinding` for this Role with our Azure AD _AKS-Admin_ group. Execute the following manifest file.

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: aks-cluster-admins-grp
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
- apiGroup: rbac.authorization.k8s.io
  kind: Group
  name: "<aad-group-object-id>"
```

```powershell
# Apply the ClusterRoleBinding to the AKS
kubectl apply --filename ".\create-group-cluster-admin-binding.yml"
```

## I). Test Access to the AKS Cluster
---

### i). Scenario 01 - No Users in the AKS-Admin Group on Azure AD

```powershell
# View Kubernetes config file
kubectl config view

# clean the current user credenials
kubectl config unset users.<user-name-from-kube-config>

# Pull down AKS credentials from Azure. No Admin
az aks get-credentials --resource-group "aks-security-rg" --name "<aks-name>"

# Execute cluster operation. 
kubectl get nodes
```

You will be prompted to use the Azure AD device login to authenticate. But once uou do login, you should see an error message like this.

`Error from server (Forbidden): nodes is forbidden: User "johnd@<your-default-dir-name>.onmicrosoft.com" cannot list nodes at the cluster scope`

This is because your user does not have access to AKS yet. You can execute following to see if you have access.

```powershell
# e.g kubectl auth can-i <command>
kubectl auth can-i get nodes
```

### ii). Scenario 02 - Add the Previously Created User to AKS-Admin Group

Let's added the _John Doe_ user to the _AKS-Admin_ AAD Group. After adding the user.

```powershell
# View Kubernetes config file
kubectl config view

# clean the current user credenials
kubectl config unset users.<user-name-from-kube-config>

# Pull down AKS credentials from Azure. No Admin
az aks get-credentials --resource-group "aks-security-rg" --name "<aks-name>"

# Execute cluster operation. 
kubectl get nodes
```

You will be prompted to use the Azure AD device login to authenticate. And now you will be able to get the nodes for the AKS cluster.

```powershell
# e.g kubectl auth can-i <command>
kubectl auth can-i get nodes
```