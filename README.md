## Simple guide to install spinnaker

### step0:
Install kubectl (ignore if already installed)
```bash
$ curl -LO https://storage.googleapis.com/kubernetes-release/release/$(curl -s https://storage.googleapis.com/kubernetes-release/release/stable.txt)/bin/linux/amd64/kubectl
$ chmod +x ./kubectl
$ sudo mv ./kubectl /usr/local/bin/kubectl
```

### step1:
Install halyard
```bash
$ curl -O https://raw.githubusercontent.com/spinnaker/halyard/master/install/debian/InstallHalyard.sh
$ echo $USER | sudo bash InstallHalyard.sh
$ sudo update-halyard # optional
```

### step2:
Set provider ()
```bash
hal config provider kubernetes enable
```

### step3:
Add k8s account to spinnaker
```bash
CONTEXT=$(kubectl config current-context)
ACCOUNT=my-k8s-v2-account
hal config provider kubernetes account add $ACCOUNT \
    --provider-version v2 \
    --context $CONTEXT
hal config features edit --artifacts true
```
### step4:
Location to install spinnaker
if installing on current system which must meet requirement  (min 4GB RAM and 2 CPU cores
```bash
$ hal config deploy edit --type localdebian
```
or if installing on same k8s cluster
```bash
$ hal config deploy edit --type distributed --account-name $ACCOUNT
```
Note: $ACCOUNT from step 3

### step5:
add docker registry
```bash
$ hal config provider docker-registry enable
$ ADDRESS="index.docker.io"
$ REPOSITORIES="library/nginx wsmathias9/node "
$ hal config provider docker-registry account add my-docker-registry \
        --address $ADDRESS \
        --repositories $REPOSITORIES
```

### step6:
set spinnaker data store to s3
```bash
$ hal config storage edit --type s3
$ hal config storage s3 edit \
    --access-key-id $YOUR_ACCESS_KEY_ID \
    --secret-access-key \
    --region $REGION
    --bucket <bucket-name>
```

### step7:
set spinnaker version.
```bash
$ hal version list
## select appropriate version from above result
VERSION=1.14.9 ## example
hal config version edit --version $VERSION
```
### step8:
install spinnaker
```bash
sudo hal deploy apply
```

### step9:
expose port to access spinnaker (k8s)
```sh
sudo hal deploy connect
```

Note: uninstall spinnaker using `hal deploy clean'

## connect via ssh tunnel

```sh
ssh -N -i key.pem -L 9000:127.0.0.1:9000 -L 8084:127.0.0.1:8084 -L 8087:127.0.0.1:8087 -L 8080:127.0.0.1:8080 ubuntu@xx.xx.xx.xx
```
