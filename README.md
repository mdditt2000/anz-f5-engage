# ANZ-F5-Engage
 
This repo is created for the ANZ F5 Engage event and documents the better together solution - **Delivering Container Ingress with F5 Container Ingress Services and NGINX Kubernetes Ingress Controller** This document is also an addition to Better together - F5 Container Ingress Services and NGINX [devcentral](https://github.com/F5Networks/k8s-bigip-ctlr/releases/tag/v2.2.3)

## Introduction

The F5 Container Ingress Services (CIS) can be integrated with the NGINX Plus Ingress Controllers (NIC) within a Kubernetes (k8s) environment.

The benefits are getting the best of both worlds, with the BIG-IP providing comprehensive L4 ~ L7 security and DNS services, while leveraging NGINX Plus as the de facto standard for micro services solution.

This architecture is depicted below

![architecture](https://github.com/mdditt2000/anz-f5-engage/raw/main/diagram/2021-02-18_14-28-09.png)

The integration is made fluid via the CIS, a k8s pod that listens to events in the cluster and dynamically populates the BIG-IP pool pointing to the NGINX IC's as they scale. CIS also dynamically populates external DSN with the Wide-IP for allowing for scaling of K8S Cluster or Data Centers.

There are a few components need to be configured to support this integration, each of which is discussed in detail.

**Step 1:**

### Create the Proxy Protocol iRule on Bigip

Proxy Protocol is required by NGINX to provide the applications PODs with the original client IPs. Use the following steps to configure the Proxy_Protocol_iRule

* Login to BigIp GUI 
* On the Main tab, click Local Traffic > iRules.
* Click Create.
* In the Name field, type name as "Proxy_Protocol_iRule".
* In the Definition field, Copy the definition from "Proxy_Protocol_iRule" file. Click Finished.

Proxy_Protocol_iRule [repo](https://github.com/mdditt2000/kubernetes-1-19/blob/master/cis%202.3/ingresslink/big-ip/proxy-protocal/irule)

**Step 2**

### Install the F5 CIS Controller 

Add BIG-IP credentials as Kubernetes Secrets

    kubectl create secret generic bigip-login -n kube-system --from-literal=username=admin --from-literal=password=<password>

Create a service account for deploying CIS.

    kubectl create serviceaccount bigip-ctlr -n kube-system

Create a Cluster Role and Cluster Role Binding on the Kubernetes Cluster as follows:
    
    kubectl create clusterrolebinding k8s-bigip-ctlr-clusteradmin --clusterrole=cluster-admin --serviceaccount=kube-system:k8s-bigip-ctlr
    
Create IngressLink Custom Resource definition as follows:

    kubectl create -f customresourcedefinition.yaml

cis-crd-schema [repo](https://github.com/mdditt2000/anz-f5-engage/blob/main/big-ip/crd-example/schema/customresourcedefinitions.yaml)

Update the bigip address, partition and other details(image, imagePullSecrets, etc) in CIS deployment file and Install CIS Controller in ClusterIP mode as follows:

* Add the following statements to the CIS deployment arguments

    - "--custom-resource-mode=true"

* To deploy the CIS controller in cluster mode update CIS deployment arguments as follows for kubernetes.

    - "--pool-member-type=cluster"
    - "--flannel-name=fl-vxlan"

Additionally, if you are deploying the CIS in Cluster Mode you need to have following prerequisites. For more information, see [Deployment Options](https://clouddocs.f5.com/containers/latest/userguide/config-options.html#config-options)
    
* You must have a fully active/licensed BIG-IP. SDN must be licensed. For more information, see [BIG-IP VE license support for SDN services](https://support.f5.com/csp/article/K26501111).
* VXLAN tunnel should be configured from Kubernetes Cluster to BIG-IP. For more information see, [Creating VXLAN Tunnels](https://clouddocs.f5.com/containers/latest/userguide/cis-helm.html#creating-vxlan-tunnels)

```
kubectl create -f f5-cis-deployment.yaml
```

cis-deployment [repo](https://github.com/mdditt2000/anz-f5-engage/blob/main/big-ip/cis-deployment/f5-cis-deployment.yaml)

Configure BIG-IP as a node in the Kubernetes cluster. This is required for OVN Kubernetes using ClusterIP

    kubectl create -f f5-bigip-node.yaml

bigip-node [repo](https://github.com/mdditt2000/anz-f5-engage/blob/main/big-ip/cis-deployment/f5-bigip-node.yaml)

Verify CIS deployment

    [kube@k8s-1-19-master cis-deployment]$ kubectl get pods -n kube-system
    NAME                                                       READY   STATUS    RESTARTS   AGE
    k8s-bigip-ctlr-deployment-fd86c54bb-w6phz                  1/1     Running   0          41s

You can view the CIS logs using the following

**Note** CIS log level is currently set to DEBUG. This can be changed in the CIS controller arguments 

    kubectl logs -f deploy/k8s-bigip-ctlr-deployment -n kube-system | grep --color=auto -i '\[debug'

**Step 3**

### Nginx-Controller Installation

Create a namespace and a service account for the Ingress controller:
   
    kubectl apply -f nginx-config/ns-and-sa.yaml
   
Create a cluster role and cluster role binding for the service account:
   
    kubectl apply -f nginx-config/rbac.yaml
   
Create a secret with a TLS certificate and a key for the default server in NGINX:

    kubectl apply -f nginx-config/default-server-secret.yaml
    
Create a config map for customizing NGINX configuration:

    kubectl apply -f nginx-config/nginx-config.yaml
    
Create an IngressClass resource (for Kubernetes >= 1.18):  
    
    kubectl apply -f nginx-config/ingress-class.yaml

Use a Deployment. When you run the Ingress Controller by using a Deployment, by default, Kubernetes will create one Ingress controller pod.
    
    kubectl apply -f nginx-config/nginx-ingress.yaml
  
Create a service for the Ingress Controller pods for ports 80 and 443 as follows:

    kubectl apply -f nginx-config/nginx-service.yaml

nginx-config [repo](https://github.com/mdditt2000/anz-f5-engage/tree/main/nginx-config)

Verify NGINX-Ingress deployment

```
[kube@k8s-1-19-master nginx-config]$ kubectl get pods -n nginx-ingress
NAME                             READY   STATUS    RESTARTS   AGE
nginx-ingress-744d95cb86-xk2vx   1/1     Running   0          16s
```

**Step 4** 

### Create an Ingress CRD Resource on BIG-IP for NGINX IC

Create a passthrough CRD and add the Ingress CRD with the Proxy Protocol iRule which is created in Step 1. This virtual server IP will be used to configure the BIG-IP device to load balance among the Ingress Controller pods.

    kubectl create -f passthrough-tls-cafe.yaml
    kubectl create -f vs-cafe.yaml

Note: The name of the app label selector in nginx-ingress resource should match the labels of the nginx-ingress service created in step-3.

crd-example [repo](https://github.com/mdditt2000/anz-f5-engage/tree/main/big-ip/crd-example/cafe)

**Step 5**

### Deploy the Cafe Application

Create the coffee and the tea deployments and services:

    kubectl create -f cafe.yaml


### Configure Load Balancing for the Cafe Application

Create a secret with an SSL certificate and a key:

    kubectl create -f cafe-secret.yaml


Create an Ingress resource:

    kubectl create -f cafe-ingress.yaml

demo application [repo](https://github.com/mdditt2000/anz-f5-engage/tree/main/ingress-example)

**Step 6**

### Test the Application

1. To access the application, curl the coffee and the tea services. We'll use ```curl```'s --insecure option to turn off certificate verification of our self-signed
certificate and the --resolve option to set the Host header of a request with ```cafe.example.com```
    
To get coffee:

    $ curl --resolve cafe.example.com:$IC_HTTPS_PORT:$IC_IP https://cafe.example.com:$IC_HTTPS_PORT/coffee --insecure
    Server address: 10.12.0.18:80
    Server name: coffee-7586895968-r26zn

If your prefer tea:

    $ curl --resolve cafe.example.com:$IC_HTTPS_PORT:$IC_IP https://cafe.example.com:$IC_HTTPS_PORT/tea --insecure
    Server address: 10.12.0.19:80
    Server name: tea-7cd44fcb4d-xfw2x


Get the `cafe-ingress` resource to check its reported address:

    $ kubectl get ing cafe-ingress
    NAME           HOSTS              ADDRESS         PORTS     AGE
    cafe-ingress   cafe.example.com   35.239.225.75   80, 443   115s

As you can see, the Ingress Controller reported the BIG-IP IP address (configured in NginxCisConnector resource) in the ADDRESS field of the Ingress status.

**Step 7**

### Configure ExternalDNS

ExternalDNS CRD's allows user to control DNS records dynamically via Kubernetes resources in a DNS provider-agnostic way. This will allow you to dynamically load balance multiple sites or kubernetes clusters

#### Setup Options

CIS provides the following options for using CIS

Add the following parameters to both CIS deployment. In this document both LTM and GTM are installed on the same device

* --custom-resource-mode=true - Configure CIS to only monitor CRDs. CIS will ignore all other resources
* --gtm-bigip-username - Provide username for CIS to access GTM
* --gtm-bigip-password - Provide password for CIS to access GTM
* --gtm-bigip-url - Provide url for CIS to access GTM. CIS uses the python SDK to configure GTM 

cis-deployment [repo](https://github.com/mdditt2000/anz-f5-engage/blob/main/big-ip/cis-deployment/f5-cis-deployment.yaml)

* Create a GSLB Servers

    - `big-ip-60-cluster

* Enable Virtual Server Discovery on BIG-IP DNS
* Deploy EDNS CRD 

    kubectl create -f edns-cafe.yaml

edns crd [repo](https://github.com/mdditt2000/anz-f5-engage/blob/main/big-ip/crd-example/cafe/edns-cafe.yaml)