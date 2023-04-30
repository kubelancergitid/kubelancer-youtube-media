#Kubernetes Expose Service with TLS using  Nginx Ingress on GCP GKE

#Pre-Req:
Create Kubernetes Cluster on GCP (Google Cloud)
Follow Video How to create Cluster
Link: https://youtu.be/iZZSpKktqps

#Let's Start

##Step 1: Install the NGINX Ingress controller
Link to Download: https://kubernetes.github.io/ingress-nginx/deploy/#gce-gke

##Step 2: Deploy the example application
kubectl run nginx --image=nginx --port=80 --label=app=nginx
kubectl create service clusterip nginx --tcp=80:80

##Step 3: Create records on  Domain
Create A record on your domain manager

##Step 4: Install cert-manager
Link: https://cert-manager.io/docs/installation/kubernetes/

##Step 5: Configure TLS with Let's Encrypt certificates and cert-manager

####Step 5.1: Create Issuer

apiVersion: cert-manager.io/v1alpha2
kind: Issuer
metadata:
  name: letsencrypt-prod
spec:
  acme:
    # The ACME server URL
    server: https://acme-v02.api.letsencrypt.org/directory
    # Email address used for ACME registration
    email: kubelancer@gmail.com
    # Name of a secret used to store the ACME account private key
    privateKeySecretRef:
      name: letsencrypt-prod
    # Enable the HTTP-01 challenge provider
    solvers:
    - http01:
       ingress:
         class: nginx



#####Step 5.2: Create Ingress

apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: nginx-ingress-tls
  namespace: default
  annotations:
    kubernetes.io/ingress.class: "nginx"
    cert-manager.io/issuer: letsencrypt-prod
spec:
  tls:
  - secretName: nginx-ingress-tls
    hosts:
    - web1.kubelancer.net
  rules:
  - host: web1.kubelancer.net
    http:
      paths:
      - path: /
        backend:
          serviceName: nginx
          servicePort: 80

##Step 6: Access the service on browser https://web1.kubelancer.net
