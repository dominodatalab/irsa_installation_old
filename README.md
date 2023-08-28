Domsed/IRSA Installation


## Domsed

If domsed is already installed, skip to the "IRSA for Domino" section 

### Install Domsed
```shell
cd domsed
export platform_namespace=domino-platform
export compute_namespace=domino-compute
helm install -f values.yaml domsed helm/domsed -n ${platform_namespace}
kubectl label namespace ${compute_namespace} operator-enabled=true
```


### Install Domsed
```shell
export platform_namespace=domino-platform
export compute_namespace=domino-compute
helm delete domsed -n ${platform_namespace}
kubectl label namespace ${compute_namespace} "operator-enabled"-
```

### Test Domsed

### Tail the logs
```
kubectl -n ${platform_namespace} get pods | grep operator
## Example output
operator-webhook-767cfcfddc-rh685                            1/1     Running    
kubectl -n ${platform_namespace} logs operator-webhook-767cfcfddc-rh685 -f
```
### Smoke Test

- Create this mutation object 
```shell
cat <<EOF | kubectl apply -f -
apiVersion: apps.dominodatalab.com/v1alpha1
kind: Mutation
metadata:
  name: label
  namespace: domino-platform
rules:
- # Insert label
  modifyLabel:
    key: "foo.com/bar"
    value: "out"
EOF
```

- Create this pod in the compute namespace
```shell
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: busybox
  namespace: domino-compute
  labels:
    dominodatalab.com/hardware-tier-id: medium-k8s  
    dominodatalab.com/project-name: quick-start
    dominodatalab.com/project-owner-username: sameer-wadkar
    dominodatalab.com/starting-user-username: sameer-wadkar
spec:
  containers:
    - name: foo
      image: busybox:unstable
EOF
```

- Verify the pod has a new label
```shell
kubectl -n ${compute_namespace} describe pod busybox
```

- Delete the pod has a new label
```shell
kubectl -n ${compute_namespace} delete pod busybox
```

## IRSA For Domino

Create a namespace domino-field

```shell
kubectl create ns domino-field
kubectl label namespace domino-field  domino-compute=true
```


### Delete IRSA

```shell
helm delete irsa -n domino-field
```

### Install IRSA

a. First create organizations in domino matching the role names

b. Update the `values.yaml` with the proper values

```shell
export eks_aws_account=<eks_aws_account>
export assets_aws_account=<assets_aws_account>
export eks_service_role_name=<eks_service_role_name>
export oidc_provider=<oidc_provider>

export platform_namespace=domino-platform
export compute_namespace=domino-compute
export field_namespace=domino-field

#Update the values.yaml with the above values
cd irsa
helm install -f ./values.yaml -n ${field_namespace} irsa helm/irsa
```

d. Copy the `irsa-certs` secret from the `domino-field` namespace to the `domino-compute` namespace
```shell
kubectl get secret irsa-certs --namespace=domino-field -o yaml | sed 's/namespace: .*/namespace: domino-compute/' | kubectl apply -f -
```
This allows the IRSA service to become SSL enabled and invokable from the workloads in the `domino-compute` namespace

## Create Mappings

Open the notebook [enablement.ipynb](./enablement.ipynb). There is a section called `## Add/Update Role Mapping (Only Domino Administrators can make this call)`

This section is used to map Domino Organizations to AWS Roles (and AWS proxy roles)
```python
EKS_ACCOUNT_NO="<ADD>"
ASSETS_ACCOUNT_NO="<ADD"
#Fetch my mappings (Any user can do this)
import requests
import os
access_token_endpoint='http://localhost:8899/access-token'
resp = requests.get(access_token_endpoint)


token = resp.text
headers = {
             "Content-Type": "application/json",
             "Authorization": "Bearer " + token,
        }


endpoint='https://irsa-svc.domino-field/update_role_mapping'
body={
    "domino_org":"irsa-iamrole-list-bucket",
    "iam_role_arn":f"arn:aws:iam::{ASSETS_ACCOUNT_NO}:role/acme-list-bucket-role",
    "proxy_iam_role_arn":f"arn:aws:iam::{EKS_ACCOUNT_NO}:role/acme-list-bucket-role"
}
resp = requests.post(endpoint,headers=headers,json=body,verify=False)
body={
    "domino_org":"irsa-iamrole-read-bucket",
    "iam_role_arn":f"arn:aws:iam::{ASSETS_ACCOUNT_NO}:role/acme-read-bucket-role",
    "proxy_iam_role_arn":f"arn:aws:iam::{EKS_ACCOUNT_NO}:role/acme-read-bucket-role"
}
resp = requests.post(endpoint,headers=headers,json=body,verify=False)
body={
    "domino_org":"irsa-iamrole-update-bucket",
    "iam_role_arn":f"arn:aws:iam::{ASSETS_ACCOUNT_NO}:role/acme-update-bucket-role",
    "proxy_iam_role_arn":f"arn:aws:iam::{EKS_ACCOUNT_NO}:role/acme-update-bucket-role"
}
resp = requests.post(endpoint,headers=headers,json=body,verify=False)

```

Walk through this notebook to get an end to end walkthrough on how to use IRSA for Domino