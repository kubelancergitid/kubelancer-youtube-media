Creating users on OpenShift 4

At first time installed openshift cluster in my AWS cloud account, I had bit confused about this error message  which appeared on top of cluster dashboard.
Then I started to research how to add users on OpenShift Container Platform. Found OpenShift OAuth server can be configured to use a number of identity providers.
On that HTPasswd identity providers, which helps to managing users and passwords against a secret that stores credentials generated using
the htpasswd. In this post how to creating users on OpenShift 4 using HTPasswd identity providers.

1. Configure an HTPasswd identity provider in OpenShift Container Platform 4.x

1.1.  Create an HTPasswd file by installing the htpasswd utility by installing the httpd-tools package on Linux

# yum install httpd-tools

1.2. Create user name and hashed password

# htpasswd -c -B -b </path/to/users.htpasswd> <user_name> <password>
# htpasswd -c -B -b users.htpasswd  ocadmin OCAdmPass
# htpasswd  -B -b users.htpasswd  devuser OCDevPass

2. Creating the HTPasswd Secret
Let we wanna to use the HTPasswd identity provider, we must define a secret that contains the HTPasswd user file.
Let create an OpenShift Container Platform Secret that contains the HTPasswd users file

# oc create secret generic htpass-secret --from-file=htpasswd=</path/to/users.htpasswd> -n openshift-config
# oc create secret generic htpass-secret --from-file=htpasswd=users.htpasswd -n openshift-config

3. Create a custom resource for an HTPasswd identity provider
# vi htpasswd.cr

apiVersion: config.openshift.io/v1
kind: OAuth
metadata:
  name: cluster
spec:
  identityProviders:
  - name: my_htpasswd_provider
    challenge: true
    login: true
    mappingMethod: claim
    type: HTPasswd
    htpasswd:
      fileData:
        name: htpass-secret

3.1. Apply the defined CR

# oc apply -f htpasswd.cr


Now the identityProviders is configured and password also added to the cluster and additionally mapped users with my_htpasswd_provider.


4. Now login using newly created user from your identity provider  and enter the password when prompted.

# oc login -u ocadmin

5. Confirm that the user logged in successfully by using below command

# oc whoami

6. Time for Authorization

6.1. Forbidden Error
# oc get nodes

Since no role is assigned to user ocadmin, this user does not have privileges to access cluster resources.

6.2. Let's configure cluster role and rolebinding

Configure cluster admin role for ocadmin
# oc adm policy add-cluster-role-to-user cluster-admin ocadmin --rolebinding-name=cluster-admin

6.3. As kubeadmin create new project
# oc new-project dev-project1

configure only specific project access for devuser
# oc adm policy add-role-to-user cluster-admin devuser -n dev-project1

6.3. Test the user ocadmin access

# oc login -u ocadmin
# oc get nodes
# oc get projects

6.4. Test the user devuser access
# oc login -u devuser
# oc get nodes
# oc get projects

6.5. Log in as the ocadmin / kubeadmin user and list the current users and identity.

# oc get users
# oc get identity


7. How to delete the user (devuser)

To delete user
# oc delete user devuser

Delete the identity for the user
# oc delete identity my_htpasswd_provider:devuser
