# ANZ-F5-Engage
 
This repo is created for the ANZ F5 Engage event and documents the better together solution - **Delivering Container Ingress with F5 Container Ingress Services and NGINX Kubernetes Ingress Controller** This document is also an addition to Better together - F5 Container Ingress Services and NGINX [devcentral](https://github.com/F5Networks/k8s-bigip-ctlr/releases/tag/v2.2.3)

## Introduction

The F5 Container Ingress Services (CIS) can be integrated with the NGINX Plus Ingress Controllers (NIC) within a Kubernetes (k8s) environment.

The benefits are getting the best of both worlds, with the BIG-IP providing comprehensive L4 ~ L7 security and DNS services, while leveraging NGINX Plus as the de facto standard for micro services solution.

This architecture is depicted below

![architecture](https://github.com/mdditt2000/anz-f5-engage/raw/main/diagram/2021-02-18_14-28-09.png)

The integration is made fluid via the CIS, a k8s pod that listens to events in the cluster and dynamically populates the BIG-IP pool pointing to the NGINX IC's as they scale. CIS also dynamically populates external DSN with the Wide-IP for allowing for scaling of K8S Cluster or Data Centers.

There are a few components need to be configured to support this integration, each of which is discussed in detail.

## NGINX Plus Ingress Controller

Follow this (https://docs.nginx.com/nginx-ingress-controller/installation/building-ingress-controller-image/) to build the NIC image.

The NIC can be deployed using the Manifests either as a Daemon-Set or a Service. See this ( https://docs.nginx.com/nginx-ingress-controller/installation/installation-with-manifests/ ).

A sample Deployment file used for this engage can be located 
