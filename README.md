## Simple guide to install spinnaker

### Step 0:
#### Install kubectl (ignore if already installed)
```bash
$ curl -LO https://storage.googleapis.com/kubernetes-release/release/$(curl -s https://storage.googleapis.com/kubernetes-release/release/stable.txt)/bin/linux/amd64/kubectl
$ chmod +x ./kubectl
$ sudo mv ./kubectl /usr/local/bin/kubectl
```

### Step 1:
#### Install halyard
```bash
$ curl -O https://raw.githubusercontent.com/spinnaker/halyard/master/install/debian/InstallHalyard.sh
$ echo $USER | sudo bash InstallHalyard.sh
$ sudo update-halyard # optional
```

### Step 2:
#### Set provider
```bash
$ hal config provider kubernetes enable
```

### Step 3(a):
### create k8s account service account for spinnaker.
````bash
$ CONTEXT=$(kubectl config current-context)

# This service account uses the ClusterAdmin role
$ kubectl apply --context $CONTEXT \
    -f https://spinnaker.io/downloads/kubernetes/service-account.yml

$ TOKEN=$(kubectl get secret --context $CONTEXT \
   $(kubectl get serviceaccount spinnaker-service-account \
       --context $CONTEXT \
       -n spinnaker \
       -o jsonpath='{.secrets[0].name}') \
   -n spinnaker \
   -o jsonpath='{.data.token}' | base64 --decode)

$ kubectl config set-credentials ${CONTEXT}-token-user --token $TOKEN
$ kubectl config set-context $CONTEXT --user ${CONTEXT}-token-user
````
### role bindings for spinnaker account
````bash
$ cat <<EOF | kubectl apply -f -
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
 name: spinnaker-role
rules:
- apiGroups: [""]
  resources: ["namespaces", "configmaps", "events", "replicationcontrollers", "serviceaccounts", "pods/log"]
  verbs: ["get", "list"]
- apiGroups: [""]
  resources: ["pods", "services", "secrets"]
  verbs: ["create", "delete", "deletecollection", "get", "list", "patch", "update", "watch"]
- apiGroups: ["autoscaling"]
  resources: ["horizontalpodautoscalers"]
  verbs: ["list", "get"]
- apiGroups: ["apps"]
  resources: ["controllerrevisions", "statefulsets"]
  verbs: ["list"]
- apiGroups: ["extensions", "apps"]
  resources: ["deployments", "replicasets", "ingresses"]
  verbs: ["create", "delete", "deletecollection", "get", "list", "patch", "update", "watch"]
# These permissions are necessary for halyard to operate. We use this role also to deploy Spinnaker itself.
- apiGroups: [""]
  resources: ["services/proxy", "pods/portforward"]
  verbs: ["create", "delete", "deletecollection", "get", "list", "patch", "update", "watch"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
 name: spinnaker-role-binding
roleRef:
 apiGroup: rbac.authorization.k8s.io
 kind: ClusterRole
 name: spinnaker-role
subjects:
- namespace: spinnaker
  kind: ServiceAccount
  name: spinnaker-service-account
---
apiVersion: v1
kind: ServiceAccount
metadata:
 name: spinnaker-service-account
 namespace: spinnaker
EOF
````

### step3(b):
#### Add k8s account to spinnaker
```bash
$ CONTEXT=$(kubectl config current-context)
$ ACCOUNT=${CONTEXT}-account
$ hal config provider kubernetes account add $ACCOUNT \
    --provider-version v2 \
    --context $CONTEXT
$ hal config features edit --artifacts true
```
#### Note: Incase of multiple clusters repeate 3(a) and 3(b) for each cluster by switcing context.

### step4:
#### Location to install spinnaker
if installing on current system which must meet requirement  (min 4GB RAM and 2 CPU cores
```bash
$ hal config deploy edit --type localdebian
```
or if installing on same k8s cluster
```bash
$ hal config deploy edit --type distributed --account-name $ACCOUNT
```
Note: $ACCOUNT is from step 3(b), choose on which cluster to install spinnaker

### step5:
#### Add docker registry
```bash
$ hal config provider docker-registry enable
$ ADDRESS="index.docker.io"
$ REPOSITORIES="library/nginx wsmathias9/node "
$ hal config provider docker-registry account add my-docker-registry \
        --address $ADDRESS \
        --repositories $REPOSITORIES
```

### step6:
#### Setup spinnaker data store to s3
```bash
$ hal config storage s3 edit \
    --access-key-id $YOUR_ACCESS_KEY_ID \
    --secret-access-key \
    --region <aws-region> \
    --bucket <bucket-name>
$ hal config storage edit --type s3
```

### step7:
#### Select spinnaker version.
```bash
$ hal version list
## select appropriate version from above result
$ VERSION=1.14.9 ## example
$ hal config version edit --version $VERSION
```
### step8:
#### Install spinnaker
```bash
$ sudo hal deploy apply
```

### step9:
#### Expose port to access spinnaker (k8s)
```sh
$ sudo hal deploy connect
```

Note: uninstall spinnaker using 
````bash
$ hal deploy clean
````

## connecting via ssh tunnel

```sh
$ ssh -N -i key.pem -L 9000:127.0.0.1:9000 -L 8084:127.0.0.1:8084 ubuntu@xx.xx.xx.xx
```
