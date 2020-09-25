# Kubernetes Demo
This procedure will show you how to:
- Install k3d
- Deploy a k3s cluster via k3d
- Deploy a private load balancer via MetalLB
- Deploy GitLab via Helm
- Configure your host with PAT to expose GitLab to the outside world
- Configure GitLab Kubernetes integration for GitLab Runners

## Prerequisites

- Tested on Ubuntu 20.04 with 16GB of RAM (referred to as workstation)
- [Docker](https://www.digitalocean.com/community/tutorials/how-to-install-and-use-docker-on-ubuntu-20-04) must be installed on your workstation
  - Your docker storage driver should be **overlay2**
    - run ```docker info | grep "Storage Driver"``` to verify
    - If your file system is XFS, be sure [this](https://github.com/rancher/k3s/issues/495) issue does not apply
- Curl must be installed on your workstation
- Nano (or similar) text editor

## Install k3d

Download and run the k3d install script

```
curl -s https://raw.githubusercontent.com/rancher/k3d/main/install.sh | bash
```

## Install kubectl

Download the latest version of kubectl

```
curl -LO "https://storage.googleapis.com/kubernetes-release/release/$(curl -s https://storage.googleapis.com/kubernetes-release/release/stable.txt)/bin/linux/amd64/kubectl"
```

### Make the kubectl binary executable

```
chmod +x ./kubectl
```

### Move the binary in to your PATH

```
sudo mv ./kubectl /usr/local/bin/kubectl
```

### Test kubectl

```
kubectl version --client
```

## Create a new cluster

Create a new k3s Kubernetes cluster with two worker nodes

```
k3d cluster create --api-port 6550 --agents 2
```

### Verify

Make sure we can access the new cluster via kubectl

```
kubectl cluster-info
kubectl get pods -A
```

At this point, you now have a fully containerized k3s deployment.  
All pods should have a status of Running.  

If not, review the logs of your master node by running:  
```docker logs k3d-k3s-default-server-0```

---

You can pause here, a play around with the cluster via kubectl.  
Once you're ready, continue on to deploy MetalLB and the GitLab suite.

---

## MetalLB

Deploy MetalLB to your cluster

```
kubectl apply -f https://raw.githubusercontent.com/google/metallb/v0.8.3/manifests/metallb.yaml
```

### Configure

Get the address space used for publishing by MetalLB  

```
kubectl get svc -n kube-system | grep traefik | awk '{ print $4 }'
```

Test the load balancer

```
curl <IP returned from previous command>
<Should return 404 - Not Found>
```

Create the a new ConfigMap for MetalLB

```
nano metal-lb-layer2-config.yaml
```

Copy/Paste the following config below, and update the addresses to match the range of MetalLB above.  
Example: If MetalLB us using 172.30.0.5, then update the addresses to be 172.30.0.5-172.30.0.100

```
apiVersion: v1
kind: ConfigMap
metadata:
  namespace: metallb-system
  name: config
data:
  config: |
    address-pools:
    - name: default
      protocol: layer2
      addresses:
      - <starting IP>-<ending IP>
```

Deploy the ConfigMap

```
kubectl create -f metal-lb-layer2-config.yaml
```

At this point, you should now have MetalLB provisioned and ready to handle new service requests.  
When you deploy GitLab, a new service will be created and you will see it grab the first available `<External>` IP from the range provided.

---

## GitLab

At this point, we're now ready to install GitLab on your k3s cluster.

### Deploy
  
**Note**: Be sure to update `global.hosts.domain` and `certmanager-issuer.email` settings before running them.

```
helm repo add gitlab https://charts.gitlab.io/
helm repo update

helm upgrade --install gitlab gitlab/gitlab \
  --timeout 800s \
  --set global.edition=ce \
  --set global.hosts.domain=<domain.com> \
  --set certmanager-issuer.email=<your.email@domain.com> \
  --set gitlab-runner.runners.privileged=true
```

### Check Deploy Status

Run the following command to check the status of the deployment.  

```
kubectl get pods
```

Please make sure all pods are in the Running status before moving on.  
This can take anywhere from 5 - 10 minutes.  

**Note:** svclb-gitlab-nginx-ingress-controller-`*` will remaining in Pending status

### Configure External Access

Now that GitLab is running, we need to configure our workstation to allow external access to GitLab.  
We do this, so external hosts can access the cluster.

#### Get Service External IP

Run the following command to get the EXTERNAL-IP for the GitLab Ingress Controller.

```
kubectl get service gitlab-nginx-ingress-controller
...
NAME                              TYPE           CLUSTER-IP     EXTERNAL-IP   PORT(S)                                   AGE
gitlab-nginx-ingress-controller   LoadBalancer   10.43.16.145   172.30.0.16   80:30640/TCP,443:30005/TCP,22:30993/TCP   16h
```

In this case, the \<Service External IP> would be `172.30.0.16`

---

At this point, you should be able to test access to GitLab via MetalLB.  
Update your workstation hostfile to point gitlab.`<domain.com>` at the Service External IP.  

Then browse to to the following URL from your workstation:  
http://`gitlab.<domain.com>`  


Once you're ready, proceed with the following steps to NAT access to GitLab.  
This will allow other hosts on your network to access the cluster. 

---

#### Get Interface Information

Run the following command to find the K8 bridge interface used by MetalLB, and the IP address of your workstation.

```
ifconfig
...
br-7322a6e7d8d3: flags=4419<UP,BROADCAST,RUNNING,PROMISC,MULTICAST>  mtu 1500
        inet 172.30.0.1  netmask 255.255.0.0  broadcast 172.30.255.255
        ...
ens33: flags=4419<UP,BROADCAST,RUNNING,PROMISC,MULTICAST>  mtu 1500
        inet 10.10.0.210  netmask 255.255.255.0  broadcast 10.10.0.255
        ...
```

In this case the:  
\<K8 Bridge Interface> would be `br-7322a6e7d8d3`  
\<Host IP> would be `10.10.0.210`

#### Configure NAT

Permit access to your cluster, and create a PAT to publish TCP ports 80 and 443.

```
sudo iptables -I FORWARD -o <K8 Bridge Interface Name> -d  <Service External IP> -j ACCEPT
sudo iptables -t nat -I PREROUTING -d <Service External IP> -p tcp --dport 80 -j DNAT --to <Host IP>:80
sudo iptables -t nat -I PREROUTING -d <Service External IP> -p tcp --dport 443 -j DNAT --to <Host IP>:443
```

---

At this point, you should be able to access GitLab from devices on your local network.  
If you wanted to, you could publish this to the web for external testing on the go.

---

## Configure K8 Integration

Now that GitLab is deployed, we can access it and update its configuration.

### Get Root Password

Use the following command to retrieve the password created for the root user.

```
kubectl get secret gitlab-gitlab-initial-root-password -ojsonpath='{.data.password}' | base64 --decode ; echo
```

### Login to GitLab

Browse to `https://gitlab.<domain.com>`

### Permit local network access

Using GitLabs navigation bar, we need to get to the Admin network settings.  
Reference: `https://gitlab.<domain.com>/admin/application_settings/network`  

Once located, under Outbound requests, you need to check both "Allow requests to local network" settings. 

Using GitLabs navigation bar, we need to get to the Admin Kubernetes settings.  
Reference: `https://gitlab.<domain.com>/admin/clusters`  

Click: Add Kubernetes Cluster  
Click: Connect existing cluster  

### Gather the requested information


Get the cluster name.

```
k3d cluster list
```

Get the name of a webservice pod.  

```
kubectl get pods | grep gitlab-webservice
```

Get the API address if your local cluster

```
kubectl exec -it <gitlab-webservice pod name> -- printenv | grep KUBE
```

Find the name of the default token for the cluster.

```
kubectl get secrets
```

Get the CA certificate for GitLab

```
kubectl get secret <defaukt-token> -o jsonpath="{['data']['ca\.crt']}" | base64 --decode
```

Create a new manifest called `gitlab-admin-service-account.yaml`, and add the follwoing to it:

```
apiVersion: v1
kind: ServiceAccount
metadata:
  name: gitlab
  namespace: kube-system
---
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: ClusterRoleBinding
metadata:
  name: gitlab-admin
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
  - kind: ServiceAccount
    name: gitlab
    namespace: kube-system

```

Create a GitLab service account

```
kubectl apply -f gitlab-admin-service-account.yaml
```

Get the service account secret

```
kubectl -n kube-system describe secret $(kubectl -n kube-system get secret | grep gitlab | awk '{print $1}')
```

---

## References:
- [K3D](https://k3d.io/)
- [Helm FAQs](https://helm.sh/docs/faq/)
- [GitLab Helm Deployment](https://docs.gitlab.com/charts/installation/deployment.html)
- [GitLab Helm Options](https://docs.gitlab.com/charts/installation/command-line-options.html)
- [K3S/K3D Deployment](https://medium.com/@lukejpreston/local-kubernetes-development-a14ea8be54d6)
- [K8 Dev Options](https://docs.tilt.dev/choosing_clusters.html)
- [Deploy MetalLB](https://blog.kubernauts.io/k3s-with-k3d-and-metallb-on-mac-923a3255c36e)
- [Publishing MetalLB](https://medium.com/better-programming/how-to-expose-your-services-with-kubernetes-ingress-7f34eb6c9b5a)
- [Kubectl Cheat Sheet](https://unofficial-kubernetes.readthedocs.io/en/latest/user-guide/kubectl-cheatsheet/)

