# How to expose dcos-ee MKE kubernetes dashboard via nginx ingress

DC/OS MKE kubernetes enterprise installation suggests to expose kubernetes dash-bord via API server as it is the best practice to remain dashboard safely accessiable over Internet. Never trying to expose kubenetes-dashboard directly over Internet as of current kubernete 1.12 release. Although it introduces TLS  and RBAC to protect anouymous access but there is still possiblility to be hacked which inadvertently expose your information. 

On the other hand, if your kubernetes is deployed under DataCenter, it really is helpful to direclty access kubernetes-dashboard without proxy through API server. 

Let's first see how can we access kubernetes-dashboard via API server. 


## 1.  access kubernetes-dashboard via kubernetes api server

To access kubernetes-dashboard via api server is straight forward, only if you have ```kubectl``` installed in your accessing PC which is inconvenient for a **NORMAL** user. 


1. First, run ```kubectl proxy``` to enable kubectl local proxy mode. 
```
$> kubectl proxy
Starting to serve on 127.0.0.1:8001
```
2. Open your Browser (Prefer **Chrome**, I got some issue with IE that can't upload my access *token* or *kubeconfig* file)

```
http://127.0.0.1:8001/api/v1/namespaces/kube-system/services/https:kubernetes-dashboard:/proxy/
```

The login page requires you to key in either **kubeconfig** or a **access token**. You could also simply skip upload credentials but you may not be able to browse any useful information without ROOT credential. 

# 2 Expose kubernetes-dashboard via Ingress Controller. 

To expose kubernetes-dashboard via Ingress may greatly improve UX to a user as oppose to kube proxy mode. 

DC/OS MKE deployment guide doesn't give a detailed instruction on how to make it happen although it shows some suggestions. 

In the article, I will leverage Nginx Ingress Controller to expose kubernetes-dashboard. 


In order to expose kubernetes-dashboard via ingress controller, you shall make sure there is at least one **public kube node** provisoned in your dc/os cluster as suggested in below


```
kube-node-public-0-kubelet.kubernetes-cluster1.mesos      Ready    <none>   5h13m   v1.12.3
```

1. To deploy Nginx Ingress Controller

```
alias k=kubectl
```
```
k create -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/master/deploy/namespace.yaml

k create -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/master/deploy/rbac.yaml

k create -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/master/deploy/with-rbac.yaml

```
You could simply create a nginx service by feed k8s with below yaml but I prefer to add my selected NodePort, like 30443 or 30080 rather than let NodePort service to select a random Node Port for me. 

```
curl -OL https://raw.githubusercontent.com/kubernetes/ingress-nginx/master/deploy/provider/baremetal/service-nodeport.yaml
```
```
apiVersion: v1
kind: Service
metadata:
  name: ingress-nginx
  namespace: ingress-nginx
  labels:
    app.kubernetes.io/name: ingress-nginx
    app.kubernetes.io/part-of: ingress-nginx
spec:
  type: NodePort
  ports:
    - name: http
      port: 80
      targetPort: 80
      nodePort: 30080 
      protocol: TCP
    - name: https
      port: 443
      targetPort: 443
      protocol: TCP
      nodePort: 30443
  selector:
    app.kubernetes.io/name: ingress-nginx
    app.kubernetes.io/part-of: ingress-nginx

---

k -f ./service-nodeport.yaml

```

The last step is crafting a HTTPS aware Ingress Rules to pass through and route dashboard request to kubernetes-dashboard. 

```
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  annotations:
    ingress.kubernetes.io/ssl-passthrough: "true"
    nginx.org/ssl-backends: "kubernetes-dashboard"
    kubernetes.io/ingress.allow-http: "false"
    nginx.ingress.kubernetes.io/backend-protocol: "HTTPS"
    nginx.ingress.kubernetes.io/service-upstream: "true"
  name: dashboard-ingress
  namespace: kube-system
spec:
  tls:
  - hosts:
    - kubernete-dashboard.example.com
    secretName: kubernetes-dashboard-certs
  rules:
  - host: kubernete-dashboard.example.com
    http:
      paths:
      - path: /
        backend:
          serviceName: kubernetes-dashboard
          servicePort: 443
```

I am using HostName Ingress Rule but probably you could also use host path rule to avoid carry hostname like **kubernete-dashboard.example.com** in your browser. 

In order to carry hostname, I manually add a entry in my own **/etc/hosts** file. 

```
##
# Host Database
#
# localhost is used to configure the loopback interface
# when the system is booting.  Do not change this entry.
##
127.0.0.1	localhost

34.209.10.136 kubernete-dashboard.example.com
```

You could capture your own **kube-node-public** node public ip via below command. 

```
dcos task | grep kube-node-public
The attached cluster is running DC/OS 1.13 but this CLI only supports DC/OS 1.12.
kube-node-public-0-kubelet                     10.0.7.54    root     R    kubernetes-cluster1__kube-node-public-0-kubelet__e99412c5-cba7-48a8-ace8-26c743d5638c            e6f8b24b-7da1-49c9-89ef-1293b46e0f8d-S0  aws/us-west-2  aws/us-west-2b
```
```
dcos node ssh --master-proxy --mesos-id=e6f8b24b-7da1-49c9-89ef-1293b46e0f8d-S0 'curl ifconfig.io'
The attached cluster is running DC/OS 1.13 but this CLI only supports DC/OS 1.12.
Running `ssh -A -t  -l core 54.202.89.44 -- ssh -A -t  -l core 10.0.7.54 -- curl ifconfig.io`
34.209.10.136
Connection to 10.0.7.54 closed.
Connection to 54.202.89.44 closed.
```

With all set, you should be able to access kubernetes-dashboard directly with your **Chrome** without enable kube proxy mode. 


```
https://kubernete-dashboard.example.com:30443/
```

# 3 Use Ingress Path rule to avoid carrying hostname/URL

If you prefer to access kubernetes-dashboard with kube-node-pub public IP without a host url like below. 


```
https://34.209.10.136:30443/
```


you shall create a Ingress Host path rule captured in below. 

```
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  annotations:
    ingress.kubernetes.io/ssl-passthrough: "true"
    nginx.org/ssl-backends: "kubernetes-dashboard"
    kubernetes.io/ingress.allow-http: "false"
    nginx.ingress.kubernetes.io/backend-protocol: "HTTPS"
    nginx.ingress.kubernetes.io/service-upstream: "true"
  name: dashboard-ingress
  namespace: kube-system
spec:
  tls:
  - hosts:
    - kubernete-dashboard.example.com
    secretName: kubernetes-dashboard-certs
  rules:
  #- host: kubernete-dashboard.example.com
  -  http:
      paths:
      - path: /
        backend:
          serviceName: kubernetes-dashboard
          servicePort: 443
```
