# TEQUILA Kubernetes Walkthrough

## Idea for the tequila session (20 minutes)

Show that it is possible to setup a Kubernetes cluster in 20 minutes.

* Setup nodes
* Join nodes to a cluster
* Ingress and SSL certificates

Quite a challenge, so let's get started. Because setting up the cluster costs some time,
I will run some commands before explaining them, then explain while running.

## Motivation

Why Kubernetes?
Because more and more projects will run on Kubernetes infrastructure
and some may need to get started quickly without waiting for IT departments.

Why do I do this?
Because from my experience once a k8s cluster is up and running, it is the best way to provision and run almost any piece of software and I want to use it in private projects.
Also my next project at Debtvision will run on Kubernetes and I have to learn.

Why in tequila?
Because DevOps is work done by the development team and part of the implementation.

__Disclaimer:__
I am very, very far from being a Kubernetes or "DevOps" expert. Some of the information shared here may be wrong or sub-optimal!
This setup is only meant to help developers getting started with a development environment.

## Let's go

First: Prove that nothing is set-up yet, except a domain and DNS for the cluster ingress.

* https://tequila.fullstackdeveloper.de
* dig tequila.fullstackdeveloper.de
* https://console.hetzner.cloud/projects/643461/servers

## Create machines in Hetzner cloud

### Setup of primary node

Create machines using cloud-init script and explain the script and what's happening.

I am using Hetzner cloud* because it allows running a very cheap cluster (~6 €/month/node = 18 €/month for a very simple cluster of 3 nodes, 4GB RAM each, shared CPU, probably not suitable for production). I have not really validated it, but I think on GKE, AWS ea, such setup would cost ~100 €/month.

    hcloud server create --name tequila01 --image ubuntu-20.04 --type cx21 --user-data-from-file cloud-init-microk8s.yaml --ssh-key mattanja@gft-homeoffice

Cloud init script:

```yaml
#cloud-config

package_update: true
package_upgrade: true
packages: ['snapd']

runcmd:
  # Floating IP address that can be assigned to the servers in Hetzner cloud
  - sudo ip addr add 78.46.255.252 dev eth0
  - sudo snap install microk8s --channel=1.19/stable --classic
  - sudo microk8s status --wait-ready
  - sudo microk8s enable dashboard dns registry ingress prometheus
```

SSH to root@new server

    ssh root@XXX
    tail -f /var/log/cloud-init-output.log

Create second and third machine

    hcloud server create --name tequila02 --image ubuntu-20.04 --type cx21 --user-data-from-file cloud-init-microk8s.yaml --ssh-key mattanja@gft-homeoffice
    hcloud server create --name tequila03 --image ubuntu-20.04 --type cx21 --user-data-from-file cloud-init-microk8s.yaml --ssh-key mattanja@gft-homeoffice

**--> Walk through Kubernetes overview while waiting**

### Join the nodes

```bash
    # Join second node (tequila02)
    microk8s add-node
    --> execute the join on other node
    # Join third node (tequila03)
    microk8s add-node
    --> execute the join on other node
    # Validate nodes and check "High-availability" (multiple master nodes and replicated datastore)
    microk8s kubectl get nodes
    microk8s status
```

**--> Continue with Ingress and Cert-Manager**

### Kubernetes overview (5 minutes)

#### Basics of Kubernetes

* Nodes
* Pods
* Ingress

Kubernetes clusters can be setup with any cloud provider in all kinds of variations.
One of the biggest problems I have with Kubernetes is the endless overwhelming amount of choices for everything.

#### Setup choices

* Environment (Google, Amazon, Azure, VMs anywhere, ...) -> Hetzner (because cheap, existing account and cloud-init support)
* Basic setup
* Installation (microk8s, kubeadm, manually, Rancher, kubespray, GKE, provider specific) --> ?? (microk8s because it is very simple and takes away all the choices I don't want to make)
* Cluster layout (Single-node, single head node with multiple workers, multiple head nodes with HA and multiple workers, HA everything) --> Going with single head node and multiple workers
* Networking (Calico, Flannel, Kube-Router, Romana, Weave Net) --> ?? (because ?)
    * _"The network must allow container-to-container, pod-to-pod, pod-to-service, and external-to-service communications."_
* Version (stable, next) --> latest stable, continuously updating every 3 months
* Logging (graylog, fluentd, kibana, grafana, ...)

#### Pitfalls

* not enough memory
* networking
* many others... Installation sounds very simple, and it is, but may take some more time to get it right.

**--> Go back to nodes setup**

## Ingress and Cert-Manager

### Setup local access

    # On node get config
    microk8s config > config
    # Copy to local
    scp root@tequila.fullstackdeveloper.de:~/config ~/.kube/config
    # Test in local terminal
    kubectl get nodes

### Setup Cert-Manager

SSL certificates are mandatory nowadays and luckily cert-manager and Let's Encrypt make this very simple.
Getting a Certificate and refreshing it every <90 days comes out-of-the-box with this setup.

    # Install from github
    kubectl apply -f https://github.com/jetstack/cert-manager/releases/download/v1.1.0/cert-manager.yaml
    # Validate
    kubectl get pods --namespace cert-manager

    # Provide letsencrypt issuer (custom config)
    kubectl apply -f setup/issuer.yaml

### Ingress Setup

    # Provide ingress for the k8s dashboard
    kubectl apply -f setup/dashboard-ingress.yaml
    # Validate
    kubectl -n kube-system describe ingress

We can now access the Kubernetes Dashboard with a fresh and valid SSL certificate https://dashboard.tequila.fullstackdeveloper.de

    # Get token for dashboard login from node (other authentication methods should be set up)
    token=$(kubectl -n kube-system get secret | grep default-token | cut -d " " -f1)
    kubectl -n kube-system describe secret $token

### Deploy our first app including ingress

    kubectl apply -f setup/tequila.yaml

## Wrap-up

We have now installed a HA Kubernetes cluster in 20 minutes, but a lot of work is to be done.

### Uncovered topics

Obviously it is not possible to cover every topic of setting up a production-ready Kubernetes Cluster in 20 minutes, but this can be a start.
Topics not covered here that will be needed next:

* Certificate handling
* Permissions (RBAC), namespaces, users, how to access the Kubernetes cluster etc...
* Volumes handling - very complex topic. If possible, keep the running components stateless and handle databases and file handling outside of the Kubernetes cluster.
* Ingress high-availability - this setup will fail if the first node fails
* Internal networking / DNS of the nodes (for example getting logs from other node fails)
* Backups
* Upgrade handling - every 3 months Kubernetes gets a new minor version with upgrade path, but shouldn't be skipped

## Additional info

Find this documentation under https://guthub.com/mattanja/tequila-microk8s

### Resources

* https://www.hetzner.com/cloud
* Google GKE Cluster in GCP Free Tier: https://cloud.google.com/free - worker nodes have to be paid for
* https://community.hetzner.com/tutorials/install-kubernetes-cluster
* https://ubuntu.com/blog/what-can-you-do-with-microk8s
  **Key takeaways and resources on MicroK8s**
  * Production-ready, fully compatible K8s on rails with selected add-ons
  * MicroK8s is available for Linux, Windows and macOS
  * Use it for dev & testing, clusters, DevOps, AI/ML, IoT, Edge and appliances
