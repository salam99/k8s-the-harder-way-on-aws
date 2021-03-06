# k8s-the-harder-way-on-aws
Based on https://github.com/rmenn/kubernetes-the-hard-way-aws, which is also based on
https://github.com/kelseyhightower/kubernetes-the-hard-way but in AWS

## Prerequisites and Conventions
- You have access to an AWS account. In my case on the us-west-2 region (cheapest, really)
- You have  enough rights. In my case, that meant IAM user with admin right, and a key/secret setup on .aws/credentials
<br>  In my case, I am calling the profile test-k8s, no wonder you'll see this often in my commands
- You have awscli setup. In my case 'pip install awscli --upgrade --user' did the trick

## Cloud provisioning
Built on top of https://github.com/rmenn/kubernetes-the-hard-way-aws/blob/master/docs/01-infra.md
Based on https://github.com/kelseyhightower/kubernetes-the-hard-way/blob/master/docs/01-infrastructure-gcp.md

### Create and tag the VPC
```
aws --profile=test-k8s ec2 create-vpc --cidr-block 10.4.0.0/16
```
Take the vpc id from the output, then:
```
aws --profile=test-k8s ec2 create-tags --resources vpc-674e9b01 --tags Key=Name,Value=afonseca-k8s-vpc
```
, where vpc-674e9b01 s the vpc ID I got from the previous step and afonseca-k8s is my project name

### Enable DNS for the VPC
```
aws --profile=test-k8s ec2 modify-vpc-attribute --vpc-id vpc-674e9b01 --enable-dns-support
aws --profile=test-k8s ec2 modify-vpc-attribute --vpc-id vpc-674e9b01 --enable-dns-hostnames
```

### Create and tag subnets for masters and workers
```
aws --profile=test-k8s ec2 create-subnet --vpc-id vpc-674e9b01 --cidr-block 10.4.1.0/24
aws --profile=test-k8s ec2 create-tags --resources subnet-4ce1072a --tags Key=Name,Value=afonseca-k8s-subnet-masters
aws --profile=test-k8s ec2 create-subnet --vpc-id vpc-674e9b01 --cidr-block 10.4.2.0/24
aws --profile=test-k8s ec2 create-tags --resources subnet-e5e50383 --tags Key=Name,Value=afonseca-k8s-subnet-workers
```
, where subnet-4ce1072a and subnet-e5e50383 are the result of the previous comman on each case

### Create and attach internet gateway
```
aws --profile=test-k8s ec2 create-internet-gateway
aws --profile=test-k8s ec2 create-tags --resources igw-eba4228c --tags Key=Name,Value=afonseca-k8s-internet-gateway
aws --profile=test-k8s ec2 attach-internet-gateway --internet-gateway-id igw-eba4228c --vpc-id vpc-674e9b01
```

### Create and config Route Tables
```
aws --profile=test-k8s ec2 create-route-table --vpc-id vpc-674e9b01
aws --profile=test-k8s ec2 create-tags --resources rtb-afb0e9c9 --tags Key=Name,Value=afonseca-k8s-route-table
```
, where rtb-afb0e9c9 is the route table id we got from the previous command
```
aws --profile=test-k8s ec2 associate-route-table --route-table-id rtb-afb0e9c9 --subnet-id subnet-4ce1072a
aws --profile=test-k8s ec2 associate-route-table --route-table-id rtb-afb0e9c9 --subnet-id subnet-e5e50383
aws --profile=test-k8s ec2 create-route --route-table-id rtb-afb0e9c9 --destination-cidr-block 0.0.0.0/0 --gateway-id igw-eba4228c
```
, where igw-eba4228c is the internet gateway id you received from the related step

### Create and config Security Groups and rules
```
aws --profile=test-k8s ec2 create-security-group --vpc-id vpc-674e9b01 --group-name afonseca-k8s-sg-masters --description afonseca-k8s-security-group-masters
aws --profile=test-k8s ec2 create-security-group --vpc-id vpc-674e9b01 --group-name afonseca-k8s-sg-workers --description afonseca-k8s-security-group-workers
aws --profile=test-k8s ec2 create-tags --resources sg-727dc908 --tags Key=Name,Value=afonseca-k8s-sg-masters
aws --profile=test-k8s ec2 create-tags --resources sg-757eca0f --tags Key=Name,Value=afonseca-k8s-sg-workers

aws --profile=test-k8s ec2 authorize-security-group-ingress --group-id sg-727dc908 --port 0-65535 --protocol tcp --source-group sg-757eca0f
aws --profile=test-k8s ec2 authorize-security-group-ingress --group-id sg-727dc908 --port 22 --protocol tcp --cidr 01.02.03.04/32
aws --profile=test-k8s ec2 authorize-security-group-ingress --group-id sg-757eca0f --port 0-65535 --protocol tcp --source-group sg-727dc908
aws --profile=test-k8s ec2 authorize-security-group-ingress --group-id sg-757eca0f --port 22 --protocol tcp --cidr 01.02.03.04/32

```
, where sg-727dc908 and sg-757eca0f are the security group IDs we got from the two first commands, respectively, and 01.02.03.04 is your local IP address (http://www.whatsmyip.org/)

The following ports are needed for etcd and etcdctl to work as a cluster:
```
aws --profile=test-k8s ec2 authorize-security-group-ingress --group-id sg-727dc908 --port 2379 --protocol tcp --source-group sg-727dc908
aws --profile=test-k8s ec2 authorize-security-group-ingress --group-id sg-727dc908 --port 2380 --protocol tcp --source-group sg-727dc908
```

### Provision the machines
```
aws --profile=test-k8s ec2 create-key-pair --key-name afonseca-k8s-key
```
, and copy the contents of KeyMaterial into ~/.ssh/afonseca-k8s-key.priv

Now do the following three times for the masters (afonseca-k8s-master01-3):
```
aws --profile=test-k8s ec2 run-instances --image-id ami-835b4efa --instance-type t2.small --key-name afonseca-k8s-key --security-group-ids sg-727dc908 --subnet-id subnet-4ce1072a --associate-public-ip-address
```
, where sg-727dc908 is the masters security group ID you got from a previous step
, and subnet-4ce1072a is the masters subnet ID you got from a previous step
```
aws --profile=test-k8s ec2 create-tags --resources i-0526417e4384cc4cc --tags Key=Name,Value=afonseca-k8s-master01
```
, where i-0526417e4384cc4cc is the InstanceID you get from the previous step...

...and do the following another three times for the workers (afonseca-k8s-worker01-3):
```
aws --profile=test-k8s ec2 run-instances --image-id ami-835b4efa --instance-type t2.small --key-name afonseca-k8s-key --security-group-ids sg-757eca0f --subnet-id subnet-e5e50383 --associate-public-ip-address
```
, where sg-757eca0f is the masters security group ID you got from a previous step
, and subnet-e5e50383 is the masters subnet ID you got from a previous step
```
aws --profile=test-k8s ec2 create-tags --resources i-02e8db8ca2d52c7ac --tags Key=Name,Value=afonseca-k8s-worker01
```
, where i-0526417e4384cc4cc is the InstanceID you get from the previous step.

### Setup Kubernetes API Server Frontend Load Balancer

ATTENTION: This has been changed from the origina because we are going to use the ELB A-Record in the following step

Create a Load Balancer for these machines:

```
aws --profile=test-k8s elb create-load-balancer --load-balancer-name afonseca-k8s-elb --listeners "Protocol=TCP,LoadBalancerPort=6443,InstanceProtocol=TCP,InstancePort=6443" --subnets subnet-4ce1072a
```
, where subnet-4ce1072a is the previously created masters-subnet id. Take note of the ELB DNSName you get (*)

Configure the health check for this ELB
```
aws --profile=test-k8s elb configure-health-check --load-balancer-name afonseca-k8s-elb --health-check Target=HTTP:8080/healthz,Interval=30,UnhealthyThreshold=2,HealthyThreshold=2,Timeout=3
```

, where afonseca-k8s-elb is the Load Balancer we just created above

Now register the master instances in the ELB:
```
aws --profile=test-k8s elb register-instances-with-load-balancer --load-balancer-name afonseca-k8s-elb --instances  i-0526417e4384cc4cc i-09c0a207c7cb864e4 i-031e5cec319b82094
```

, where i-0526417e4384cc4cc i-09c0a207c7cb864e4 i-031e5cec319b82094 are the restult of running:
```
aws --profile=test-k8s ec2 describe-instances --filters "Name=subnet-id,Values=subnet-4ce1072a" | grep "InstanceId"
```

Check the new ELB's security group
TODO: maybe it's better to force sg when we define the ELB?
```
aws --profile=test-k8s elb describe-load-balancers --load-balancer-name afonseca-k8s-elb | grep -a1 SecurityGroups
```

Then open the ports needed
```
aws --profile=test-k8s ec2 authorize-security-group-ingress --group-id sg-727dc908 --port 8080 --protocol tcp --source-group sg-a2c276d8
```


## Setup CA and create TLS certs

### Install CFSSL

```
wget https://pkg.cfssl.org/R1.2/cfssl_linux-amd64
chmod +x cfssl_linux-amd64
sudo mv cfssl_linux-amd64 /usr/local/bin/cfssl

wget https://pkg.cfssl.org/R1.2/cfssljson_linux-amd64
chmod +x cfssljson_linux-amd64
sudo mv cfssljson_linux-amd64 /usr/local/bin/cfssljson
```

### Set up a Certificate Authority

#### Create a CA configuration file:
```
cat > ca-config.json <<EOF
{
  "signing": {
    "default": {
      "expiry": "8760h"
    },
    "profiles": {
      "kubernetes": {
        "usages": ["signing", "key encipherment", "server auth", "client auth"],
        "expiry": "8760h"
      }
    }
  }
}
EOF
```

#### Create a CA certificate signing request:
```
cat > ca-csr.json <<EOF
{
  "CN": "Kubernetes",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "US",
      "L": "Portland",
      "O": "Kubernetes",
      "OU": "CA",
      "ST": "Oregon"
    }
  ]
}
EOF
```

### Generate a CA certificate and private key:

```
cfssl gencert -initca ca-csr.json | cfssljson -bare ca

```
### Generate client and server TLS certificates

#### Create the Admin client certificate
```
cat > admin-csr.json <<EOF
{
  "CN": "admin",
  "hosts": [],
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "US",
      "L": "Portland",
      "O": "system:masters",
      "OU": "Cluster",
      "ST": "Oregon"
    }
  ]
}
EOF
```

#### Generate the admin client certificate and private key:
```
cfssl gencert \
  -ca=ca.pem \
  -ca-key=ca-key.pem \
  -config=ca-config.json \
  -profile=kubernetes \
  admin-csr.json | cfssljson -bare admin
```

### Create the kube-proxy client certificate

#### Create the kube-proxy client certificate signing request:
```
cat > kube-proxy-csr.json <<EOF
{
  "CN": "system:kube-proxy",
  "hosts": [],
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "US",
      "L": "Portland",
      "O": "system:node-proxier",
      "OU": "Cluster",
      "ST": "Oregon"
    }
  ]
}
EOF
```

#### Generate the kube-proxy client certificate and private key:
```
cfssl gencert \
  -ca=ca.pem \
  -ca-key=ca-key.pem \
  -config=ca-config.json \
  -profile=kubernetes \
  kube-proxy-csr.json | cfssljson -bare kube-proxy
```

#### Create the kubernetes server certificate

The original uses an EIP, but we are going to use the ELB DNS instead
```
KUBERNETES_PUBLIC_ADDRESS="afonseca-k8s-elb-1913340698.us-west-2.elb.amazonaws.com"
echo $KUBERNETES_PUBLIC_ADDRESS

cat > kubernetes-csr.json <<EOF
{
  "CN": "kubernetes",
  "hosts": [
    "10.32.0.1",
    "10.4.1.150",
    "10.4.1.77",
    "10.4.1.106",
    "${KUBERNETES_PUBLIC_ADDRESS}",
    "127.0.0.1",
    "kubernetes.default"
  ],
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "US",
      "L": "Portland",
      "O": "Kubernetes",
      "OU": "Cluster",
      "ST": "Oregon"
    }
  ]
}
EOF
```
, where 10.4.1.150 is master01, 10.4.1.77 is master02, 10.4.1.106 and...
IMPORTANT: I don't have a clue where 10.32.0.1 comes from on Kelsey Hightower's version. UPDATE: I might have some clue
IMPORTANT: make sure KUBERNETES_PUBLIC_ADDRESS gets substituted by the IP

#### Generate the Kubernetes certificate and private key:
```
cfssl gencert \
  -ca=ca.pem \
  -ca-key=ca-key.pem \
  -config=ca-config.json \
  -profile=kubernetes \
  kubernetes-csr.json | cfssljson -bare kubernetes
```

### Distribute the TLS certificates
```
for host in a b c; do scp ca.pem ca-key.pem kubernetes-key.pem kubernetes.pem ubuntu@$host:/home/ubuntu/ ; done
for host in x y z; do scp ca.pem kube-proxy.pem kube-proxy-key.pem ubuntu@$host:/home/ubuntu/ ; done
```
, where a b and c are the the masters' IPs, and x, y and z are the three workers' IPs
If you want to check those IPs you can use the AWS console or something like:
```
aws --profile=test-k8s ec2 describe-instances --query 'Reservations[*].Instances[*].[InstanceId,NetworkInterfaces[*].PrivateIpAddress,NetworkInterfaces[*].PrivateIpAddresses[*].Association.PublicIp]' | grep -v "\[\|\]"
```

## Setting up Authentication
Brutally copied over from https://github.com/kelseyhightower/kubernetes-the-hard-way/blob/master/docs/03-auth-configs.md

- Download and Install kubectl - https://kubernetes.io/docs/tasks/tools/install-kubectl/

### Authentication

The following components will leverage Kubernetes RBAC:

- kubelet (client)
- kube-proxy (client)
- kubectl (client)

The other components, mainly the scheduler and controller manager, access the Kubernetes API server locally over the insecure API port which does not require authentication.
The insecure port is only enabled for local access.

#### Create the TLS Bootstrap Token

This section will walk you through the creation of a TLS bootstrap token that will be used to bootstrap TLS client certificates for kubelets.

Generate a token:
```
BOOTSTRAP_TOKEN=$(head -c 16 /dev/urandom | od -An -t x | tr -d ' ')
```

Generate a token file:
```
cat > token.csv <<EOF
${BOOTSTRAP_TOKEN},kubelet-bootstrap,10001,"system:kubelet-bootstrap"
EOF
```

Distribute the token
```
for host in a b c; do scp token.csv ubuntu@$host:/home/ubuntu/ ; done
```
, where a b and c are the the masters' IPs.

### Client Authentication Configs

This section will walk you through creating kubeconfig files that will be used to bootstrap kubelets, which will then generate their own kubeconfigs based on dynamically generated certificates, and a kubeconfig for authenticating kube-proxy clients.

Each kubeconfig requires a Kubernetes master to connect to. To support H/A the IP address assigned to the load balancer sitting in front of the Kubernetes API servers will be used.

#### Set the kubernetes Public Address
Make sure you still have the EIP as an environment variable set:
```
KUBERNETES_PUBLIC_ADDRESS="34.211.127.220"
echo $KUBERNETES_PUBLIC_ADDRESS
```
, where 34.211.127.220 is the IP we got on a previous step (see "Create the kubernetes server certificate")

### Create client kubeconfig files

#### Create the bootstrap kubeconfig file
```
kubectl config set-cluster kubernetes-the-hard-way \
  --certificate-authority=ca.pem \
  --embed-certs=true \
  --server=https://${KUBERNETES_PUBLIC_ADDRESS}:6443 \
  --kubeconfig=bootstrap.kubeconfig
```
```
kubectl config set-credentials kubelet-bootstrap \
  --token=${BOOTSTRAP_TOKEN} \
  --kubeconfig=bootstrap.kubeconfig
```
```
kubectl config set-context default \
  --cluster=kubernetes-the-hard-way \
  --user=kubelet-bootstrap \
  --kubeconfig=bootstrap.kubeconfig
```
```
kubectl config use-context default --kubeconfig=bootstrap.kubeconfig
```

#### Create the kube-proxy kubeconfig
```
kubectl config set-cluster kubernetes-the-hard-way \
  --certificate-authority=ca.pem \
  --embed-certs=true \
  --server=https://${KUBERNETES_PUBLIC_ADDRESS}:6443 \
  --kubeconfig=kube-proxy.kubeconfig
```
```
kubectl config set-credentials kube-proxy \
  --client-certificate=kube-proxy.pem \
  --client-key=kube-proxy-key.pem \
  --embed-certs=true \
  --kubeconfig=kube-proxy.kubeconfig
```
```
kubectl config set-context default \
  --cluster=kubernetes-the-hard-way \
  --user=kube-proxy \
  --kubeconfig=kube-proxy.kubeconfig
```
```
kubectl config use-context default --kubeconfig=kube-proxy.kubeconfig
```

### Distribute the client kubeconfig files
```
for host in x y z; do scp bootstrap.kubeconfig kube-proxy.kubeconfig ubuntu@$host:/home/ubuntu/ ; done
```
, where x, y and z are your workers' IPs

## Bootstrapping a H/A etcd cluster
Based on https://github.com/kelseyhightower/kubernetes-the-hard-way/blob/master/docs/04-etcd.md

We will now setup a 3 node etcd cluster on master01, master02 and master03

### Why

All Kubernetes components are stateless which greatly simplifies managing a Kubernetes cluster. All state is stored in etcd, which is a database and must be treated specially. To limit the number of compute resource to complete this lab etcd is being installed on the Kubernetes controller nodes, although some people will prefer to run etcd on a dedicated set of machines for the following reasons:

    The etcd lifecycle is not tied to Kubernetes. We should be able to upgrade etcd independently of Kubernetes.
    Scaling out etcd is different than scaling out the Kubernetes Control Plane.
    Prevent other applications from taking up resources (CPU, Memory, I/O) required by etcd.

However, all the e2e tested configurations currently run etcd on the master nodes.

### Provision the etcd Cluster

Until further notice, Run all the following commands on all masters:

#### TLS Certificates

The TLS certificates created in the Setting up a CA and TLS Cert Generation lab will be used to secure communication between the Kubernetes API server and the etcd cluster. The TLS certificates will also be used to limit access to the etcd cluster using TLS client authentication. Only clients with a TLS certificate signed by a trusted CA will be able to access the etcd cluster.

Copy the TLS certificates to the etcd configuration directory:
```
sudo mkdir -p /etc/etcd/
sudo cp ca.pem kubernetes-key.pem kubernetes.pem /etc/etcd/
```

#### Download and Install the etcd binaries

Download the official etcd release binaries from coreos/etcd GitHub project:
```
wget https://github.com/coreos/etcd/releases/download/v3.1.4/etcd-v3.1.4-linux-amd64.tar.gz
```
Extract and install the etcd server binary and the etcdctl command line client:
```
tar -xvf etcd-v3.1.4-linux-amd64.tar.gz
sudo mv etcd-v3.1.4-linux-amd64/etcd* /usr/bin/
```
All etcd data is stored under the etcd data directory. In a production cluster the data directory should be backed by a persistent disk. Create the etcd data directory:
```
sudo mkdir -p /var/lib/etcd
```

#### Set the Internal IP Address
The internal IP address will be used by etcd to serve client requests and communicate with other etcd peers.
```
INTERNAL_IP=$(curl http://169.254.169.254/latest/meta-data/local-ipv4)
```

Each etcd member must have a unique name within an etcd cluster. Set the etcd name:
```
ETCD_NAME=master0X
```
, where X is the number of the master you are in

The etcd server will be started and managed by systemd. Create the etcd systemd unit file:
```
cat > etcd.service <<EOF
[Unit]
Description=etcd
Documentation=https://github.com/coreos

[Service]
ExecStart=/usr/bin/etcd \\
  --name ${ETCD_NAME} \\
  --cert-file=/etc/etcd/kubernetes.pem \\
  --key-file=/etc/etcd/kubernetes-key.pem \\
  --peer-cert-file=/etc/etcd/kubernetes.pem \\
  --peer-key-file=/etc/etcd/kubernetes-key.pem \\
  --trusted-ca-file=/etc/etcd/ca.pem \\
  --peer-trusted-ca-file=/etc/etcd/ca.pem \\
  --peer-client-cert-auth \\
  --client-cert-auth \\
  --initial-advertise-peer-urls https://${INTERNAL_IP}:2380 \\
  --listen-peer-urls https://${INTERNAL_IP}:2380 \\
  --listen-client-urls https://${INTERNAL_IP}:2379,http://127.0.0.1:2379 \\
  --advertise-client-urls https://${INTERNAL_IP}:2379 \\
  --initial-cluster-token etcd-cluster-0 \\
  --initial-cluster  master01=https://10.4.1.150:2380,master02=https://10.4.1.77:2380,master03=https://10.4.1.106:2380 \\
  --initial-cluster-state new \\
  --data-dir=/var/lib/etcd
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF
```
, where "master01=https://10.4.1.150:2380,master02=https://10.4.1.77:2380,master03=https://10.4.1.106:2380" has to be adapted to the IPs you have

Once the etcd systemd unit file is ready, move it to the systemd system directory:
```
sudo mv etcd.service /etc/systemd/system/
```

Start the etcd server:
```
sudo systemctl daemon-reload
sudo systemctl enable etcd
sudo systemctl start etcd
sudo systemctl status etcd --no-pager
```
## Bootstrapping an H/A Kubernetes Control Plane
Blatantly copied from https://github.com/kelseyhightower/kubernetes-the-hard-way/blob/master/docs/05-kubernetes-controller.md

Here we bootstrap a 3-node k8s controller cluster and create a frontend load balancer with a public IP address for remote access to the API servers and H/A.

### Why

The Kubernetes components that make up the control plane include the following components:

    API Server
    Scheduler
    Controller Manager

Each component is being run on the same machine for the following reasons:

    The Scheduler and Controller Manager are tightly coupled with the API Server
    Only one Scheduler and Controller Manager can be active at a given time, but it's ok to run multiple at the same time. Each component will elect a leader via the API Server.
    Running multiple copies of each component is required for H/A
    Running each component next to the API Server eases configuration.

### Provision the Kubernetes Controller Cluster

The following commands are meant to be run on all three master-0x servers. ssh-login to those machines first.

Copy the bootstrap token into place:
```
sudo mkdir -p /var/lib/kubernetes/
sudo mv token.csv /var/lib/kubernetes/
```

#### TLS Certificates

The TLS certificates created in the Setting up a CA and TLS Cert Generation lab will be used to secure communication between the Kubernetes API server and Kubernetes clients such as kubectl and the kubelet agent. The TLS certificates will also be used to authenticate the Kubernetes API server to etcd via TLS client auth.

Copy the TLS certificates to the Kubernetes configuration directory:
```
sudo mv ca.pem ca-key.pem kubernetes-key.pem kubernetes.pem /var/lib/kubernetes/
```
#### Download and install the Kubernetes controller binaries

Download the official Kubernetes release binaries:
```
wget https://storage.googleapis.com/kubernetes-release/release/v1.7.0/bin/linux/amd64/kube-apiserver
wget https://storage.googleapis.com/kubernetes-release/release/v1.7.0/bin/linux/amd64/kube-controller-manager
wget https://storage.googleapis.com/kubernetes-release/release/v1.7.0/bin/linux/amd64/kube-scheduler
wget https://storage.googleapis.com/kubernetes-release/release/v1.7.0/bin/linux/amd64/kubectl
```
Install the k8s binaries
```
chmod +x kube-apiserver kube-controller-manager kube-scheduler kubectl
sudo mv kube-apiserver kube-controller-manager kube-scheduler kubectl /usr/bin/
```
#### Kubernetes API Server

Capture the internal IP address of the machine:
```
INTERNAL_IP=$(curl http://169.254.169.254/latest/meta-data/local-ipv4)
```
Create the systemd unit file:
```
cat > kube-apiserver.service <<EOF
[Unit]
Description=Kubernetes API Server
Documentation=https://github.com/GoogleCloudPlatform/kubernetes

[Service]
ExecStart=/usr/bin/kube-apiserver \\
  --admission-control=NamespaceLifecycle,LimitRanger,ServiceAccount,DefaultStorageClass,ResourceQuota \\
  --advertise-address=${INTERNAL_IP} \\
  --allow-privileged=true \\
  --apiserver-count=3 \\
  --audit-log-maxage=30 \\
  --audit-log-maxbackup=3 \\
  --audit-log-maxsize=100 \\
  --audit-log-path=/var/lib/audit.log \\
  --authorization-mode=RBAC \\
  --bind-address=0.0.0.0 \\
  --client-ca-file=/var/lib/kubernetes/ca.pem \\
  --enable-swagger-ui=true \\
  --etcd-cafile=/var/lib/kubernetes/ca.pem \\
  --etcd-certfile=/var/lib/kubernetes/kubernetes.pem \\
  --etcd-keyfile=/var/lib/kubernetes/kubernetes-key.pem \\
  --etcd-servers=https://10.4.1.150:2379,https://10.4.1.77:2379,https://10.4.1.106:2379 \\
  --event-ttl=1h \\
  --experimental-bootstrap-token-auth \\
  --insecure-bind-address=0.0.0.0 \\
  --kubelet-certificate-authority=/var/lib/kubernetes/ca.pem \\
  --kubelet-client-certificate=/var/lib/kubernetes/kubernetes.pem \\
  --kubelet-client-key=/var/lib/kubernetes/kubernetes-key.pem \\
  --kubelet-https=true \\
  --runtime-config=rbac.authorization.k8s.io/v1alpha1 \\
  --service-account-key-file=/var/lib/kubernetes/ca-key.pem \\
  --service-cluster-ip-range=10.32.0.0/24 \\
  --service-node-port-range=30000-32767 \\
  --tls-cert-file=/var/lib/kubernetes/kubernetes.pem \\
  --tls-private-key-file=/var/lib/kubernetes/kubernetes-key.pem \\
  --token-auth-file=/var/lib/kubernetes/token.csv \\
  --v=2
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF
```
Start the kube-apiserver service:
```
sudo mv kube-apiserver.service /etc/systemd/system/
sudo systemctl daemon-reload
sudo systemctl enable kube-apiserver
sudo systemctl start kube-apiserver
sudo systemctl status kube-apiserver --no-pager
```

#### Kubernetes controller manager

```
cat > kube-controller-manager.service <<EOF
[Unit]
Description=Kubernetes Controller Manager
Documentation=https://github.com/GoogleCloudPlatform/kubernetes

[Service]
ExecStart=/usr/bin/kube-controller-manager \\
  --address=0.0.0.0 \\
  --allocate-node-cidrs=true \\
  --cluster-cidr=10.4.0.0/16 \\
  --cluster-name=kubernetes \\
  --cluster-signing-cert-file=/var/lib/kubernetes/ca.pem \\
  --cluster-signing-key-file=/var/lib/kubernetes/ca-key.pem \\
  --leader-elect=true \\
  --master=http://${INTERNAL_IP}:8080 \\
  --root-ca-file=/var/lib/kubernetes/ca.pem \\
  --service-account-private-key-file=/var/lib/kubernetes/ca-key.pem \\
  --service-cluster-ip-range=10.32.0.0/16 \\
  --v=2
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF
```
Start the kube-controller-manager service:
```
sudo mv kube-controller-manager.service /etc/systemd/system/
sudo systemctl daemon-reload
sudo systemctl enable kube-controller-manager
sudo systemctl start kube-controller-manager
sudo systemctl status kube-controller-manager --no-pager
```


#### Kubernetes scheduler
```
cat > kube-scheduler.service <<EOF
[Unit]
Description=Kubernetes Scheduler
Documentation=https://github.com/GoogleCloudPlatform/kubernetes

[Service]
ExecStart=/usr/bin/kube-scheduler \\
  --leader-elect=true \\
  --master=http://${INTERNAL_IP}:8080 \\
  --v=2
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF
```

Start the kube-scheduler service:
```
sudo mv kube-scheduler.service /etc/systemd/system/
sudo systemctl daemon-reload
sudo systemctl enable kube-scheduler
sudo systemctl start kube-scheduler
sudo systemctl status kube-scheduler --no-pager
```

### Verification

```
kubectl get componentstatuses
```
...Bootstraping the ELB was done before, at provisioning time

## Bootstrapping Kubernetes Workers

### Why

Kubernetes worker nodes are responsible for running your containers. All Kubernetes clusters need one or more worker nodes. We are running the worker nodes on dedicated machines for the following reasons:

    Ease of deployment and configuration
    Avoid mixing arbitrary workloads with critical cluster components. We are building machines with just enough resources so we don't have to worry about wasting resources.

Some people would like to run workers and cluster services anywhere in the cluster. This is totally possible, and you'll have to decide what's best for your environment.

### Prerequisites

Each worker node will provision a unique TLS client certificate as defined in the kubelet TLS bootstrapping guide. The kubelet-bootstrap user must be granted permission to request a client TLS certificate.

ssh into one of the masters and, from there, enable TLS bootstrapping by binding the kubelet-bootstrap user to the system:node-bootstrapper cluster role:
```
kubectl create clusterrolebinding kubelet-bootstrap \
  --clusterrole=system:node-bootstrapper \
  --user=kubelet-bootstrap
```

On each worker, run the following:
```
sudo mkdir -p /var/lib/{kubelet,kube-proxy,kubernetes}
sudo mkdir -p /var/run/kubernetes
sudo mv bootstrap.kubeconfig /var/lib/kubelet
sudo mv kube-proxy.kubeconfig /var/lib/kube-proxy
```

move TLS certs in place

```
sudo mv ca.pem /var/lib/kubernetes/
```

### Install Docker

```
wget https://get.docker.com/builds/Linux/x86_64/docker-1.12.6.tgz
tar -xvf docker-1.12.6.tgz
sudo cp docker/docker* /usr/bin/
```

Create the Docker systemd unit file:

```
cat > docker.service <<EOF
[Unit]
Description=Docker Application Container Engine
Documentation=http://docs.docker.io

[Service]
ExecStart=/usr/bin/docker daemon \\
  --iptables=false \\
  --ip-masq=false \\
  --host=unix:///var/run/docker.sock \\
  --log-level=error \\
  --storage-driver=overlay
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF
```


Start the docker service:
```
sudo mv docker.service /etc/systemd/system/docker.service
sudo systemctl daemon-reload
sudo systemctl enable docker
sudo systemctl start docker
sudo docker version
```

### Install the kubelet

The Kubelet can now use CNI - the Container Network Interface to manage machine level networking requirements.

Download and install CNI plugins

```
sudo mkdir -p /opt/cni
wget https://storage.googleapis.com/kubernetes-release/network-plugins/cni-amd64-0799f5732f2a11b329d9e3d51b9c8f2e3759f2ff.tar.gz
sudo tar -xvf cni-amd64-0799f5732f2a11b329d9e3d51b9c8f2e3759f2ff.tar.gz -C /opt/cni
```

Download and install the Kubernetes worker binaries:

```
wget https://storage.googleapis.com/kubernetes-release/release/v1.6.1/bin/linux/amd64/kubectl
wget https://storage.googleapis.com/kubernetes-release/release/v1.6.1/bin/linux/amd64/kube-proxy
wget https://storage.googleapis.com/kubernetes-release/release/v1.6.1/bin/linux/amd64/kubelet
chmod +x kubectl kube-proxy kubelet
sudo mv kubectl kube-proxy kubelet /usr/bin/
```

Create the kubelet systemd unit file:

```
API_SERVERS=$(sudo cat /var/lib/kubelet/bootstrap.kubeconfig | grep server | cut -d ':' -f2,3,4 | tr -d '[:space:]')
echo $API_SERVERS

cat > kubelet.service <<EOF
[Unit]
Description=Kubernetes Kubelet
Documentation=https://github.com/GoogleCloudPlatform/kubernetes
After=docker.service
Requires=docker.service

[Service]
ExecStart=/usr/bin/kubelet \\
  --api-servers=${API_SERVERS} \\
  --allow-privileged=true \\
  --cluster-dns=10.32.0.10 \\
  --cluster-domain=cluster.local \\
  --container-runtime=docker \\
  --experimental-bootstrap-kubeconfig=/var/lib/kubelet/bootstrap.kubeconfig \\
  --network-plugin=kubenet \\
  --kubeconfig=/var/lib/kubelet/kubeconfig \\
  --serialize-image-pulls=false \\
  --register-node=true \\
  --tls-cert-file=/var/lib/kubelet/kubelet-client.crt \\
  --tls-private-key-file=/var/lib/kubelet/kubelet-client.key \\
  --cert-dir=/var/lib/kubelet \\
  --v=2
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF

sudo mv kubelet.service /etc/systemd/system/kubelet.service
sudo systemctl daemon-reload
sudo systemctl enable kubelet
sudo systemctl start kubelet
sudo systemctl status kubelet --no-pager
```

### kube-proxy

Then, back to the workers, keep running the following on each one:
```
cat > kube-proxy.service <<EOF
[Unit]
Description=Kubernetes Kube Proxy
Documentation=https://github.com/GoogleCloudPlatform/kubernetes

[Service]
ExecStart=/usr/bin/kube-proxy \\
  --cluster-cidr=10.4.0.0/16 \\
  --masquerade-all=true \\
  --kubeconfig=/var/lib/kube-proxy/kube-proxy.kubeconfig \\
  --proxy-mode=iptables \\
  --v=2
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF

sudo mv kube-proxy.service /etc/systemd/system/kube-proxy.service
sudo systemctl daemon-reload
sudo systemctl enable kube-proxy
sudo systemctl start kube-proxy
sudo systemctl status kube-proxy --no-pager
```

### Approve the TLS certificate requests

Each worker node will submit a certificate signing request which must be approved before the node is allowed to join the cluster.

ssh into one of the masters and, from there, List the pending certificate requests:
```
kubectl get csr
```

--ERROR: kube-proxy not coming up kube-proxy[32624]: invalid configuration: default cluster has no server defined
--ERROR: No certs arrived, no connection through 6443 I believe
--ERROR: the EIP seems to be used for the ELB in the original case, we need to move to Route53 OR the ELB itself


# NEXTUP
- Clean up everything
- Recreate everything automatically with a script
- Solve issue with 6443 (no csr but also ELB shows it's not "healthy")

# Cleanup
- Review if IGW is still needed, as well as SSL access
