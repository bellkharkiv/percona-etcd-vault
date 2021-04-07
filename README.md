# percona-etcd-vault
 Open Issues
Vault url is http and not https
Monitoring with Prometheus 
Add node selector for each infra ETCD, Vault and Percona

Comments
Before installing the infrastructure, please, study attentively the attached files with the examples how we did settings for our infrastructure. It gives you a general vision of how it should be done on your side. Please don't hesitate to contact us if you have any questions.

Install & setup ETCD and Vault

1. Install etcd:    https://github.com/bitnami/charts/tree/master/bitnami/etcd
    $ kubectl create ns etcd
    $ helm repo add bitnami https://charts.bitnami.com/bitnami  
    $ helm install etcd bitnami/etcd -n etcd --debug \
  --set replicaCount=2 \
  --set persistence.enabled=true \
  --set persistence.size=2Gi \
  --set auth.rbac.rootPassword=YOUR_PASSWORD_FOR_ETCD \
  --set nodeSelector=<Node labels for pod assignment> \
  --set metrics.enabled=true 


Metrics parameters https://github.com/bitnami/charts/tree/master/bitnami/etcd#metrics-parameters


 helm install etcd bitnami/etcd -n etcd --debug  --set replicaCount=3 --set persistence.enabled=true --set persistence.size=2Gi --set auth.rbac.rootPassword=Test1!


Assign Pods to Nodes using Node Affinity
$ kubectl label nodes <your-node-name> label=<your-label>
$ kubectl get nodes --show-labels
 

2. Add user and KMS key to AWS
2.1 Create new user in AWS account (vault),  Access type => Programmatic access, copy Access key ID (<VAULT_USER_ACCESS_KEY>), Secret access key (<VAULT_USER_SECRET_KEY>)
2.2 Create key in kms (AWS) choice Symmetric A single encryption key that is used for both encrypt and decrypt
       operations, click next, enter Alias (name your key), click next, enter Key administrators (add user who creates key),
       click next, define key usage permissions (add the new user we created, vault) (<AWS_KMS_KEY_ID>)

3. Install Vault: https://github.com/hashicorp/vault-helm 
    $ kubectl create ns vault
    $ git clone https://github.com/hashicorp/vault-helm.git (Need to download new code from GitHub)
    $ cd vault-helm
3.1 You need to edit the values.yaml file adding parameters
NodeSelector for Vault https://learn.hashicorp.com/tutorials/vault/kubernetes-reference-architecture?in=vault/kubernetes

HA with TLS (https://www.vaultproject.io/docs/platform/k8s/helm/examples/standalone-tls)
global:

  tlsDisable: false

server:
  extraEnvironmentVars:
    VAULT_CACERT: /vault/userconfig/vault-server-tls/vault.ca

  extraVolumes:
    - type: secret
      name: vault-server-tls # Matches the ${SECRET_NAME} from above


  standalone:
    enabled: false
	…

  ha:
    enabled: true
    replicas: 2

    config: |
      listener "tcp" {
        address = "[::]:8200"
        cluster_address = "[::]:8201"
        tls_cert_file = "/vault/userconfig/vault-server-tls/vault.crt"
        tls_key_file  = "/vault/userconfig/vault-server-tls/vault.key"
        tls_client_ca_file = "/vault/userconfig/vault-server-tls/vault.ca"
      }

     telemetry {
       prometheus_retention_time = "30s"
       disable_hostname = true
     }
  
      storage "etcd" {
        address  = "http://etcd.etcd.svc:2379"
        etcd_api = "v3"
        username = "root"
        password = "YOUR_PASSWORD_FOR_ETCD"
        ha_enabled  = "true"
      }
      seal "awskms" {
        region     = "eu-central-1"
        access_key = "<VAULT_USER_ACCESS_KEY>"
        secret_key = "<VAULT_USER_SECRET_KEY>"
        kms_key_id = "<AWS_KMS_KEY_ID>"
      }
 

commands to generate SSL certificate and private key
$  SERVICE=vault
$  NAMESPACE=vault
$  SECRET_NAME=vault-server-tls
$  TMPDIR=/root/ssl/
$  openssl genrsa -out ${TMPDIR}/vault.key 2048
$  cd /root/ssl
$  cat <<EOF >${TMPDIR}/csr.conf
[req]
req_extensions = v3_req
distinguished_name = req_distinguished_name
[req_distinguished_name]
[ v3_req ]
basicConstraints = CA:FALSE
keyUsage = nonRepudiation, digitalSignature, keyEncipherment
extendedKeyUsage = serverAuth
subjectAltName = @alt_names
[alt_names]
DNS.1 = ${SERVICE}
DNS.2 = ${SERVICE}.${NAMESPACE}
DNS.3 = ${SERVICE}.${NAMESPACE}.svc
DNS.4 = ${SERVICE}.${NAMESPACE}.svc.cluster.local
IP.1 = 127.0.0.1
EOF
$  openssl req -new -key ${TMPDIR}/vault.key -subj "/CN=${SERVICE}.${NAMESPACE}.svc" -out ${TMPDIR}/server.csr -config {TMPDIR}/csr.conf
$  export CSR_NAME=vault-csr
$  cat <<EOF >${TMPDIR}/csr.yaml
apiVersion: certificates.k8s.io/v1beta1
kind: CertificateSigningRequest
metadata:
  name: ${CSR_NAME}
spec:
  groups:
  - system:authenticated
  request: $(cat ${TMPDIR}/server.csr | base64 | tr -d '\n')
  usages:
  - digital signature
  - key encipherment
  - server auth
EOF

$  kubectl create -f ${TMPDIR}/csr.yaml
$  kubectl certificate approve ${CSR_NAME}
$  serverCert=$(kubectl get csr ${CSR_NAME} -o jsonpath='{.status.certificate}')
$  echo "${serverCert}" | openssl base64 -d -A -out ${TMPDIR}/vault.crt
$  kubectl config view --raw --minify --flatten -o jsonpath='{.clusters[].cluster.certificate-authority-data}' | base64 -d > ${TMPDIR}/vault.ca
$  kubectl create secret generic ${SECRET_NAME} \
         --namespace ${NAMESPACE} \
         --from-file=vault.key=${TMPDIR}/vault.key \
         --from-file=vault.crt=${TMPDIR}/vault.crt \
         --from-file=vault.ca=${TMPDIR}/vault.ca



Prometheus monitoring vault
In the prometheus config you need to add
# prometheus.yml
scrape_configs:
  - job_name: 'vault'
    metrics_path: "/v1/sys/metrics"
    params:
      format: ['prometheus']
    scheme: https
    tls_config:
      ca_file: your_ca_here.pem
    bearer_token: "your_vault_token_here"
    static_configs:
    - targets: ['your_vault_server_here:8200']
https://www.vaultproject.io/docs/configuration/telemetry#prometheus

3.2 install Vault
    $ helm install -n vault vault ./ 
3.3 You need to initialize the Vault.
    $ kubectl get pods -n vault 
    $ kubectl exec -it vault-0 -n vault -- sh
For auto-unseal, to work, you need to make an init once with the following command
    $ vault operator init 
you will see something like this
Recovery Key 1: q+jzuRuljKTKGRD139BeSc+q878Pc4uyDoXkL+1n5nA0
Recovery Key 2: g3zyC7liaGi4ENnyWxtcGGO1QdsgIkDHQqr5XkJOLAW6
Recovery Key 3: H1F4aekCK7VQfJ6VrdBfdTTMxptqdJKnwTnR8KPLAyz/
Recovery Key 4: rYjk9RbUiAG6nabXfX+Pbps+w2FqVh+ROLJudigOvwBO
Recovery Key 5: lqhv7pJtIuZBzIke3ZStuajB6Bibs5Emx+AagGzq5Q1T

Initial Root Token: s.EAwKiixSkB16CAxbYczc8bud

Save keys and token somewhere in file!!!
Login to create mount point for percona:
    $ vault login s.EAwKiixSkB16CAxbYczc8bud
    $ vault secrets enable --version=1 -path=pxc-secret kv

exit the container 
    $ exit
Backups for Vault (optional)
Install in local machine etcdcli v3. Ubuntu 18.04:
$ apt install etcd-client
$ kubectl port-forward --address localhost pod/etcd-0 32379:2379 
$ etcdctl --debug --endpoints=http://127.0.0.1:32379 --user=root:YOUR_PASSWORD_FOR_ETCD snapshot save /tmp/snapshot-$(date +%Y%m%d).db




Install & setup Install Percona XtraDB Cluster
Installing the PMM Server (monitoring) https://www.percona.com/blog/2020/07/23/using-percona-kubernetes-operators-with-percona-monitoring-and-management/

Edit cr.yaml
...
pmm:
       enabled: true
...
$ helm repo add percona https://percona-charts.storage.googleapis.com
$ helm repo update
$ helm install monitoring percona/pmm-server --set platform=kubernetes --version 2.7.0 --set "credentials.password=<YOUR_PASSWORD>"


1. Create a namespace pxc
$ kubectl create namespace pxc
$ kubectl config set-context $(kubectl config current-context) --namespace=pxc

2. Use the following git clone command to download the correct branch of the percona-xtradb-cluster-operator repository:
$ git clone -b v1.7.0 https://github.com/percona/percona-xtradb-cluster-operator

After the repository is downloaded, change the directory to run the rest of the commands in this document:
$ cd percona-xtradb-cluster-operator

3. Edit deploy/vault-secret.yamll (https://www.percona.com/doc/kubernetes-operator-for-pxc/encryption.html#encryption) and deploy the Operator with the following command:
$ kubectl apply -f deploy/vault-secret.yaml -n pxc
$ kubectl apply -f deploy/bundle.yaml -n pxc

The following confirmation is returned:
customresourcedefinition.apiextensions.k8s.io/perconaxtradbclusters.pxc.percona.com created
customresourcedefinition.apiextensions.k8s.io/perconaxtradbclusterbackups.pxc.percona.com created
customresourcedefinition.apiextensions.k8s.io/perconaxtradbclusterrestores.pxc.percona.com created
customresourcedefinition.apiextensions.k8s.io/perconaxtradbbackups.pxc.percona.com created
role.rbac.authorization.k8s.io/percona-xtradb-cluster-operator created
serviceaccount/percona-xtradb-cluster-operator created
rolebinding.rbac.authorization.k8s.io/service-account-percona-xtradb-cluster-operator created
deployment.apps/percona-xtradb-cluster-operator created

4. The operator has been started, and you can create the Percona XtraDB cluster:
    $ kubectl apply -f deploy/cr.yaml -n pxc

The process could take some time. The return statement confirms the creation:
perconaxtradbcluster.pxc.percona.com/cluster1 created

5. During previous steps, the Operator has generated several secrets, including the password for the root user, which you will need to access the cluster.
Use kubectl get secrets command to see the list of Secrets objects (by default Secrets object you are interested in has my-cluster-secrets name). Then 
    $ kubectl get secret my-cluster-secrets -o yaml
will return the YAML file with generated secrets, including the root password which should look as follows:
...
data:
  ...
  root: cm9vdF9wYXNzd29yZA==
Here the actual password is base64-encoded, and echo 'cm9vdF9wYXNzd29yZA==' | base64 --decode will bring it back to a human-readable form (in this example it will be a root_password string).

6. Now you can check wether you are able to connect to MySQL from the outside with the help of the kubectl port-forward command as follows:
$ kubectl port-forward svc/example-proxysql 3306:3306
$ mysql -h 127.0.0.1 -P 3306 -uroot -proot_password

Run this on first setup
SET GLOBAL pxc_strict_mode=PERMISSIVE;



Backups for MySQL
Making on-demand backup: 
$ kubectl apply -f deploy/backup-s3.yaml -n pxc (credential aws)
$ kubectl apply -f backup/backup.yaml -n pxc (manual backup)
https://github.com/percona/percona-xtradb-cluster-operator/blob/main/deploy/backup/README.md




Setup for microk8s
sudo microk8s.enable dns rbac storage helm3 dns


Useful links
TLS for Vault
https://www.vaultproject.io/docs/platform/k8s/helm/examples/standalone-tls
Auto-unseal with AWS KMS:
https://learn.hashicorp.com/tutorials/vault/autounseal-aws-kms
TDE for Percona
https://www.percona.com/doc/kubernetes-operator-for-pxc/encryption.html#id3
Percona on EKS
https://www.percona.com/doc/kubernetes-operator-for-pxc/eks.html

