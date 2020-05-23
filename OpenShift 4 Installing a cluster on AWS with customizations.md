Install OpenShift Cluster 4 on AWS Cloud
##############################

Infrastructure:
----------------
Openshift version: 4.3
Cloud: AWS


Accounts Sign Up
----------------------
If you don't already have an AWS account and Red Hat account, create it first.
Sign up for a Red Hat subscription. https://www.redhat.com/wapps/ugc/register.html
Sign Up AWS account https://aws.amazon.com

Setting up jumpserver
=====================

Install the AWS CLI version 1 on Linux
---------------------------------------

yum install python3 -y
python get-pip.py --user
cp -rf .local/bin/* /usr/local/bin/
pip3 install awscli --upgrade --user
cp .local/bin/aws /usr/local/bin/ ; chmod +x /usr/local/bin/aws
aws --version

Generating an SSH private key and adding it to the agent
-------------------------------------------------------

ssh-keygen -t rsa -b 4096 -N '' -f ~/.ssh/id_rsa
eval "$(ssh-agent -s)"
ssh-add ~/.ssh/id_rsa




Configure an AWS account to host your cluster
--------------------------------------------
1. Configuring Route53
- Amazon Web Services (AWS) account you use must have a dedicated public hosted zone in your Route53 service

2. AWS Accounts limits
Ref : https://docs.openshift.com/container-platform/4.3/installing/installing_aws/installing-aws-account.html

3. Create IAM user

- Create the IAM user [ oscuser ] name and select Programmatic access.
- Attach the AdministratorAccess policy to ensure that the account has sufficient permission

4. AWS Configure
[root@ip-172-31-7-245 ~]# aws configure
AWS Access Key ID [None]: AKIAV62DSXXXXXXX
AWS Secret Access Key [None]: AcpxFM2YVJ0XXXXXXXXX
Default region name [None]: us-east-2
Default output format [None]: json
[root@ip-172-31-7-245 ~]#

Obtaining the installation program
-----------------------------------
https://cloud.redhat.com/openshift/install

Downloads
------------

1. OpenShift installer
curl -O https://mirror.openshift.com/pub/openshift-v4/clients/ocp/latest/openshift-install-linux.tar.gz
tar -xvzf openshift-install-linux.tar.gz
cp -rf openshift-install /usr/local/bin/openshift-install ; chmod +x /usr/local/bin/openshift-install

2. Pull secret
Download or copy your pull secret. The install program will prompt you for your pull secret during installation.

3. Command-line interface
curl -O https://mirror.openshift.com/pub/openshift-v4/clients/ocp/latest/openshift-client-linux.tar.gz
tar -xvzf openshift-client-linux.tar.gz
cp -rf oc /usr/local/bin/oc ; chmod +x /usr/local/bin/oc ; cp -rf kubectl /usr/local/bin/kubectl; chmod +x /usr/local/bin/kubectl

Creating the installation configuration file
--------------------------------------------
1. Create installation_directory [oscdir]
mkdir ./oscdir

2. Create the install-config.yaml file
openshift-install create install-config --dir=./oscdir

3. Modify the install-config.yaml file
vi install-config.yaml

[root@ip-172-31-7-245 oscdir]# cat install-config.yaml
apiVersion: v1
baseDomain: kubelancer.net
compute:
- architecture: amd64
  hyperthreading: Enabled
  name: worker
  platform:
    aws:
      type: t2.medium
  replicas: 2
controlPlane:
  architecture: amd64
  hyperthreading: Enabled
  name: master
  platform:
    aws:
      type: t2.medium
  replicas: 1
metadata:
  creationTimestamp: null
  name: kubecluster
networking:
  clusterNetwork:
  - cidr: 10.128.0.0/14
    hostPrefix: 23
  machineNetwork:
  - cidr: 10.0.0.0/16
  networkType: OpenShiftSDN
  serviceNetwork:
  - 172.30.0.0/16
platform:
  aws:
    region: us-east-2


Deploy the cluster
-----------------

openshift-install create cluster --dir=./oscdir  --log-level=info


Access your cluster
---------------------

You can log into your cluster as a default system user by exporting the cluster kubeconfig file:

export KUBECONFIG=<installation_directory>/auth/kubeconfig

Verify cluster
===============
oc whoami
oc cluster-info
oc version
oc get nodes


Destroy Openshift cluster
-----------------------
openshift-install destroy cluster --dir=./oscdir --log-level=info
