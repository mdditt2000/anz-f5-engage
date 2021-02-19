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

cis-crd-schema [repo](https://github.com/mdditt2000/kubernetes-1-19/blob/master/cis%202.3/ingresslink/cis/ingresslink/cis-crd-schema/ingresslink-customresourcedefinition.yaml)

Update the bigip address, partition and other details(image, imagePullSecrets, etc) in CIS deployment file and Install CIS Controller in ClusterIP mode as follows:

* Add the following statements to the CIS deployment arguments

    - "--custom-resource-mode=true"

* To deploy the CIS controller in cluster mode update CIS deploymemt arguments as follows for kubernetes.

    - "--pool-member-type=cluster"
    - "--flannel-name=fl-vxlan"

Additionally, if you are deploying the CIS in Cluster Mode you need to have following prerequisites. For more information, see [Deployment Options](https://clouddocs.f5.com/containers/latest/userguide/config-options.html#config-options)
    
* You must have a fully active/licensed BIG-IP. SDN must be licensed. For more information, see [BIG-IP VE license support for SDN services](https://support.f5.com/csp/article/K26501111).
* VXLAN tunnel should be configured from Kubernetes Cluster to BIG-IP. For more information see, [Creating VXLAN Tunnels](https://clouddocs.f5.com/containers/latest/userguide/cis-helm.html#creating-vxlan-tunnels)

```
kubectl create -f f5-cis-deployment.yaml
```

cis-deployment [repo](https://github.com/mdditt2000/kubernetes-1-19/blob/master/cis%202.3/ingresslink/cis/ingresslink/cis-deployment/f5-cis-deployment.yaml)

Configure BIG-IP as a node in the Kubernetes cluster. This is required for OVN Kubernetes using ClusterIP

    kubectl create -f f5-bigip-node.yaml

bigip-node [repo](https://github.com/mdditt2000/kubernetes-1-19/blob/master/cis%202.3/ingresslink/cis/ingresslink/cis-deployment/f5-bigip-node.yaml)

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

Verify NGINX-Ingress deployment

```
[kube@k8s-1-19-master nginx-config]$ kubectl get pods -n nginx-ingress
NAME                             READY   STATUS    RESTARTS   AGE
nginx-ingress-744d95cb86-xk2vx   1/1     Running   0          16s
```