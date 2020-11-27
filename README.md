# HashiCorp Vault as a KMS for Ceph

The following readme guides you thru the steps to set up a demo running a 1 node Rook/Ceph hosted on Minikube and a local Vault running outside of Minikube.
It also uses a Vault Agent as a side car proxy to the Rados Gateway for maximum security and production readiness.

## Install and start minikube

    $ minikube start (ou minikube start --driver=virtualbox)

When installing rook for the first time, make sure we have a raw device on the minikube host
(https://rook.io/docs/rook/v1.3/ceph-quickstart.html)

    $ minikube ssh
    $ lsblk -f
    $ exit

## Install and configure Rook

    $ git clone --single-branch --branch release-1.3 https://github.com/rook/rook.git

### Deploy the rook operator

    $ cd ./rook/cluster/examples/kubernetes/ceph
    $ kubectl create -f common.yaml
    $ kubectl create -f operator.yaml

### Verify the rook-ceph-operator is in the `Running` state before proceeding

    $ kubectl -n rook-ceph get pod

### Update the Ceph config (ceph.conf)
First, update the cluster-test.yaml file with the floowing ceph configuration:

    debug rgw = 20/5
    rgw crypt s3 kms backend = vault
    rgw crypt vault auth = agent
    rgw crypt vault addr = http://localhost:8100
    rgw crypt vault secret engine = transit
    rgw crypt vault prefix = /v1/transit/export/encryption-key
    rgw crypt require ssl = false

### Deploy the rook cluster

    $ kubectl create -f cluster-test.yaml


### Check pods are running

    $ kubectl -n rook-ceph get pod

### Check logs

    $ kubectl logs -n rook-ceph --all-containers rook-ceph-osd-prepare-minikube-dztj7

(you should see that Ceph is using a raw device to create an OSD. If it can't, you will see an error message at the bottom)


### Provision Object storage

    $ kubectl create -f object-test.yaml

### Check that the rgw pod is running (it takes a few minutes before being deployed)

    $ kubectl -n rook-ceph get pod -l app=rook-ceph-rgw

### Create a StorageClass

    $ kubectl create -f storageclass-bucket-delete.yaml

### Claim a bucket

    $ kubectl create -f object-bucket-claim-delete.yaml

### Install and run the toolbox

    $ kubectl apply -f toolbox.yaml
    $ kubectl -n rook-ceph exec -it $(kubectl -n rook-ceph get pod -l "app=rook-ceph-tools" -o jsonpath='{.items[0].metadata.name}') -- /bin/sh

### Look at the ceph and osd status

    # ceph status
    (health status is HEALTH_WARN. Because we have a single k8s node in Minikube, no replica are configured)
    # ceph osd status

### Check that the S3 bucket has been created

    # radosgw-admin bucket list
    [
        "rookbucket-6ae7d67e-0e27-42a5-ba00-bb869739b163"
    ]


## Configure Vault

### Enable the transit engine

    $ vault secrets enable transit

### Create an exportable key

    $ vault write -f transit/keys/mybucketkey exportable=true

### Create a policy and a token to access the previous key

    $ TOKEN=xxx vault read transit/export/encryption-key/mybucketkey/1

### Configure k8s AuthN method
(See https://learn.hashicorp.com/tutorials/vault/agent-kubernetes)

#### Configure Authorization for the token reviewer

    $ kubectl apply -f vault-auth-service-account.yaml

#### Configure the Vault k8s role with the right policy to access the encryption key 

    $ vault write auth/kubernetes/role/radosgwrole \
            bound_service_account_names=default \
            bound_service_account_namespaces=rook-ceph \
            policies=rgw-transit2-ro \
            ttl=24h

#### Configure the k8s authN method

    $ export VAULT_SA_NAME=$(kubectl get sa default -o jsonpath="{.secrets[*]['name']}" -n rook-ceph)
    $ export SA_JWT_TOKEN=$(kubectl get secret $VAULT_SA_NAME \
        -o jsonpath="{.data.token}" -n rook-ceph | base64 --decode; echo)
    $ export SA_CA_CRT=$(kubectl get secret $VAULT_SA_NAME \
        -o jsonpath="{.data['ca\.crt']}" -n rook-ceph | base64 --decode; echo)
    $ export K8S_HOST=$(minikube ip)
    $ vault write auth/kubernetes/config \
            token_reviewer_jwt="$SA_JWT_TOKEN" \
            kubernetes_host="https://$K8S_HOST:8443" \
            kubernetes_ca_cert="$SA_CA_CRT"

## Deploy the Vault Agent

### Deployment of the configmap for vault-agent

    $ kubectl apply -f vault-agent-config.yaml

### Patching of the rgw pod to add the vault-agent

    $ kubectl patch deploy rook-ceph-rgw-my-store-a --patch "$(cat cluster-test-patch.yaml)" -n rook-ceph

### Check that your vault-agent is up and running (and able to authenticate to Vault Server)

    $ kubectl -n rook-ceph get pod -l app=rook-ceph-rgw
    $ kubectl -n rook-ceph logs rook-ceph-rgw-my-store-a-5c5854d68c-svjn9 -c vault-agent-auth-template


## Testing: Communicate as a client to the RadosGateway

### Get you S3 AKSK

    $ kubectl -n default get secret ceph-delete-bucket -o yaml | grep AWS_ACCESS_KEY_ID | awk '{print $2}' | base64 --decode
    $ kubectl -n default get secret ceph-delete-bucket -o yaml | grep AWS_SECRET_ACCESS_KEY | awk '{print $2}' | base64 --decode

### Install and run the toolbox

    $ kubectl apply -f toolbox.yaml
    $ kubectl -n rook-ceph exec -it $(kubectl -n rook-ceph get pod -l "app=rook-ceph-tools" -o jsonpath='{.items[0].metadata.name}') -- /bin/sh

### Check that the S3 bucket has been created

    # radosgw-admin bucket list
    [
        "rookbucket-6ae7d67e-0e27-42a5-ba00-bb869739b163"
    ]

### Upload a file to the newly created bucket

    # echo "Hello Rook" > /tmp/rookObj
    # echo "Hello Rook encrypted" > /tmp/rookObj2
    # yum --assumeyes install s3cmd
    # s3cmd put /tmp/rookObj --access_key=6GYOI27BO9GWNN87EU72 --secret_key=HslhLOXztccsxEpB6VEdbGYkbDGQ3Vf7wD9Tuyao  --no-ssl --host=rook-ceph-rgw-my-store.rook-ceph --host-bucket=  s3://rookbucket-6ae7d67e-0e27-42a5-ba00-bb869739b163
    # s3cmd put /tmp/rookObj2 --access_key=6GYOI27BO9GWNN87EU72 --secret_key=HslhLOXztccsxEpB6VEdbGYkbDGQ3Vf7wD9Tuyao  --no-ssl --host=rook-ceph-rgw-my-store.rook-ceph --server-side-encryption --server-side-encryption-kms-id=mybucketkey/1 --host-bucket=  s3://rookbucket-6ae7d67e-0e27-42a5-ba00-bb869739b163

### Download and verify the file content from the bucket

    # s3cmd get s3://rookbucket-6ae7d67e-0e27-42a5-ba00-bb869739b163/rookObj /tmp/rookObj-download --access_key=6GYOI27BO9GWNN87EU72 --secret_key=HslhLOXztccsxEpB6VEdbGYkbDGQ3Vf7wD9Tuyao  --no-ssl --host=rook-ceph-rgw-my-store.rook-ceph --host-bucket=
    # cat /tmp/rookObj-download
    # s3cmd get s3://rookbucket-6ae7d67e-0e27-42a5-ba00-bb869739b163/rookObj2 /tmp/rookObj2-download --access_key=6GYOI27BO9GWNN87EU72 --secret_key=HslhLOXztccsxEpB6VEdbGYkbDGQ3Vf7wD9Tuyao  --no-ssl --host=rook-ceph-rgw-my-store.rook-ceph --host-bucket=
    # cat /tmp/rookObj2-download

### Check that my data is encrypted
(Run the toolbox)

    # ceph osd lspools
    # rados -p my-store.rgw.buckets.data ls
    # rados -p my-store.rgw.buckets.data get 00efb6b5-40ca-4812-9f80-c097de8e512f.4395.1_rookObj /tmp/notencrypted
    # cat /tmp/notencrypted
    # rados -p my-store.rgw.buckets.data get 00efb6b5-40ca-4812-9f80-c097de8e512f.4395.1_rookObj2 /tmp/encrypted


## Debugging

### Check rgw logs

    $ kubectl -n rook-ceph logs -f rook-ceph-rgw-my-store-a-688c79f8ff-5xtpv
    
### Check that your Vault key is accessible

    $ curl --header "X-Vault-Token: s.WK1AUXXXXXXXX" http://192.168.0.12:8200/v1/transit2/export/encryption-key/mybucketkey/1 | jq


    
 
## Enable the Dashboard (optional)

### Create a User

    $ kubectl create -f object-user.yaml

### Check that the user was created

    $ kubectl -n rook-ceph describe secret rook-ceph-object-user-my-store-my-user

### Retrieve AccessKey for this user

    $ kubectl -n rook-ceph get secret rook-ceph-object-user-my-store-my-user -o yaml | grep AccessKey | awk '{print $2}' | base64 --decode

### Retrieve SecretKey for this user

    $ kubectl -n rook-ceph get secret rook-ceph-object-user-my-store-my-user -o yaml | grep SecretKey | awk '{print $2}' | base64 --decode

### Access the Ceph Dashboard

    $ kubectl port-forward -n rook-ceph svc/rook-ceph-mgr-dashboard 7000:7000

### Enabling Dashboard Object Gateway management (commands to be entered in the toolbox)
#### Enable system flag on the user:

    radosgw-admin user modify --uid=my-user --system

#### Provide the user credentials:

    ceph dashboard set-rgw-api-user-id my-user
    ceph dashboard set-rgw-api-access-key <access-key>
    ceph dashboard set-rgw-api-secret-key <secret-key>


## Clean up things

    $ kubectl delete -f object-user.yaml (if created)
    $ kubectl delete -f toolbox.yaml
    $ kubectl delete -f object-bucket-claim-delete.yaml
    $ kubectl delete -f storageclass-bucket-delete.yaml
    $ kubectl delete -f object-test.yaml
    $ kubectl delete -f cluster-test.yaml
    $ kubectl delete -f operator.yaml
    $ kubectl delete -f common.yaml
    $ minikube ssh "sudo rm -rf /var/lib/rook"
    $ minikube stop
    (delete the inital raw device from virtualbox)


References:
- [1] https://rook.io/docs/rook/v1.3/ceph-quickstart.html
- [2] https://rook.io/docs/rook/v1.3/ceph-block.html
- [3] https://github.com/rook/rook/issues/5301
- [4] https://docs.ceph.com/docs/master/radosgw/vault/
- [5] https://blog.zwindler.fr/2019/09/10/du-ceph-dans-mon-kubernetes/
- [6] https://medium.com/@vovaprivalov/setup-and-playing-with-rook-storage-with-minikube-a9424ffcac4b
