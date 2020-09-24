# Kubernetes - GitLab
Documented steps to deploy a development Kubernetes (k3s/k3d) cluster on Ubuntu 20.04

# Create Cluster

Run the following command to create a new Kubernetes cluster.

```
k3d cluster create --api-port 6550 --agents 2
```

## Verify

Let's check and make sure that we can access the new cluster via kubectl.

```
kubectl cluster-info
k3d kubeconfig get -a
kubectl get pods -A
```

# MetalLB

The following commands are used to install MetalLB.  
MetalLB allows your local Kubernetes cluster to provision a LB for your external services.  
When using a cloud Kubernetes PaaS, this is handled by the cloud platform. Since we're running Kubernetes locally, we need MetalLB.

## Install

```
git clone https://github.com/arashkaffamanesh/k3d-k3s-metallb.git
cd k3d-k3s-metallb

kubectl apply -f https://raw.githubusercontent.com/google/metallb/v0.8.3/manifests/metallb.yaml
```

## Configure

```
kubectl get svc -n kube-system | grep traefik | awk '{ print $4 }'

vim metal-lb-layer2-config.yaml
# adapt the addresses field to something like this 172.20.0.3–172.20.0.254 in metal-lb-layer2-config.yaml

kubectl create -f metal-lb-layer2-config.yaml

curl 172.20.0.2
# Should return a 404 - Page not found

```

# GitLab

At this point, we're now ready to install GitLab to the Kubernetes cluster.

## Install
  
Note: Be sure to update the commands before running them.

```
helm repo add gitlab https://charts.gitlab.io/
helm repo update

helm upgrade --install gitlab gitlab/gitlab \
  --timeout 800s \
  --set global.edition=ce \
  --set global.hosts.domain=configpy.com \
  --set certmanager-issuer.email=nate.mellendorf@gmail.com \
  --set gitlab-runner.runners.privileged=true
```

## Check Deployment

Run the following command to check the status of the deployment.  
Once all pods are running, we can move on to the next step.

```
kubectl get pods
```

## Configure External Access

Now that GitLab is running, we need to configure our host to allow external access to GitLab. This is needed for testing.

### Get Service External IP

Run the following command to get the EXTERNAL-IP for the GitLab Ingress Controller.

```
kubectl get service gitlab-nginx-ingress-controller
...
NAME                              TYPE           CLUSTER-IP     EXTERNAL-IP   PORT(S)                                   AGE
gitlab-nginx-ingress-controller   LoadBalancer   10.43.16.145   172.30.0.16   80:30640/TCP,443:30005/TCP,22:30993/TCP   16h
```

In this casen the \<Service External IP> would be **172.30.0.16**

### Get Interface Information

Run the following command to find the K8 bridge interface used by MetalLB, and the IP address of the host that's running K3S/K3D.

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
\<K8 Bridge Interface> would be **br-7322a6e7d8d3**  
\<Host IP> would be **10.10.0.210**

### Configure NAT

We need to add additional IPTable rules on the host, which will allow for external access to the MetalLB IP address.  
In this case, we'll permit access to and create a NAT for HTTP and HTTPS traffic.

```
sudo iptables -I FORWARD -o <K8 Bridge Interface Name> -d  <Service External IP> -j ACCEPT
sudo iptables -t nat -I PREROUTING -d <Service External IP> -p tcp --dport 80 -j DNAT --to <Host IP>:80
sudo iptables -t nat -I PREROUTING -d <Service External IP> -p tcp --dport 443 -j DNAT --to <Host IP>:443
```

## Configure

Now that GitLab is deployed, we can access it and update its configuration.

### Get Root Password

Use the following command to retrieve the password created for the root user.

```
kubectl get secret gitlab-gitlab-initial-root-password -ojsonpath='{.data.password}' | base64 --decode ; echo
```

### Configure K8 Integration

Use the following command to get the cluster name.

```
k3d cluster list
```

Use the following to get the CA certificate for GitLab

```
kubectl get secrets
kubectl get secret default-token-9tkdm -o jsonpath="{['data']['ca\.crt']}" | base64 --decode
```

Create a new manifest called **gitlab-admin-service-account.yaml**, and add the follwoing to it.

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

Create the service account

```
kubectl apply -f gitlab-admin-service-account.yaml
```

Get the service account secret

```
kubectl -n kube-system describe secret $(kubectl -n kube-system get secret | grep gitlab | awk '{print $1}')
```

# References:
- [K3D](https://k3d.io/)
- [Helm FAQs](https://helm.sh/docs/faq/)
- [GitLab Helm Deployment](https://docs.gitlab.com/charts/installation/deployment.html)
- [GitLab Helm Options](https://docs.gitlab.com/charts/installation/command-line-options.html)
- [K3S/K3D Deployment](https://medium.com/@lukejpreston/local-kubernetes-development-a14ea8be54d6)
- [K8 Dev Options](https://docs.tilt.dev/choosing_clusters.html)
- [Deploy MetalLB](https://blog.kubernauts.io/k3s-with-k3d-and-metallb-on-mac-923a3255c36e)
- [Publishing MetalLB](https://medium.com/better-programming/how-to-expose-your-services-with-kubernetes-ingress-7f34eb6c9b5a)
- [Kubectl Cheat Sheet](https://unofficial-kubernetes.readthedocs.io/en/latest/user-guide/kubectl-cheatsheet/)

