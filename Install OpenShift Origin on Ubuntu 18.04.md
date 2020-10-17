# Install OpenShift Origin on AWS EC2 Ubuntu 18.04

## Environment:

```
Cloud: AWS
AMI: ubuntu/images/hvm-ssd/ubuntu-bionic-18.04-amd64-server
Instance type: t2.medium
VPC: default
Security groups :

Inbound Rules

80	TCP	0.0.0.0/0
22	TCP	0.0.0.0/0
8443 TCP	0.0.0.0/0
443	TCP	0.0.0.0/0
```

## Install Docker

```bash
$ sudo apt-get update

$ sudo apt-get install \
    apt-transport-https \
    ca-certificates \
    curl \
    gnupg-agent \
    software-properties-common

$ curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -

$ sudo apt-key fingerprint 0EBFCD88



$ sudo add-apt-repository \
   "deb [arch=amd64] https://download.docker.com/linux/ubuntu \
   $(lsb_release -cs) \
   stable"

$ sudo apt-get update
$ sudo apt-get install docker-ce docker-ce-cli containerd.io


$ sudo usermod -aG docker $USER
(Remember to log out and back in for this to take effect!)

```

## Install openshift origin client tool

```bash
github : https://github.com/openshift/origin/releases/tag/v3.11.0

$ sudo wget https://github.com/openshift/origin/releases/download/v3.11.0/openshift-origin-client-tools-v3.11.0-0cbc58b-linux-64bit.tar.gz
$ sudo tar -xvzf openshift-origin-client-tools-v3.11.0-0cbc58b-linux-64bit.tar.gz
$ sudo cd openshift-origin-client-tools-v3.11.0-0cbc58b-linux-64bit/
$ sudo mv oc kubectl /usr/local/bin/ ; cd ..

```
### Login as root

```bash

$ sudo su -


cat << EOF > /etc/docker/daemon.json
{
    "insecure-registries" : [ "172.30.0.0/16" ]
}
EOF

systemctl restart docker

```
## Login as ubuntu

```
AWS_HOSTNAME=`curl http://169.254.169.254/latest/meta-data/public-hostname`

sudo oc cluster up --routing-suffix=$AWS_HOSTNAME --public-hostname=$AWS_HOSTNAME

```

## Error / Workaround

```bash

sudo oc cluster down

sudo vi ./openshift.local.clusterup/openshift-controller-manager/openshift-master.kubeconfig


In that file, search for the line:

server: https://127.0.0.1:8443

Replace that line with:

server: https://<EC2 Public DNS NAME>:8443

Save and close the file. Bring the cluster back up with the command:

oc cluster up --routing-suffix=$AWS_HOSTNAME --public-hostname=$AWS_HOSTNAME

```

## output

```bash

OpenShift server started.

The server is accessible via web console at:
    https://ec2-3-17-28-122.us-east-2.compute.amazonaws.com:8443

You are logged in as:
    User:     developer
    Password: <any value>

To login as administrator:
    oc login -u system:admin

```

 # * * * * *  Happy Computing  * * * * *

 
