# How to create a user in Kubernetes with X509

One of the neccessory things you might want to do after bring up a new kubernetes cluster is to add more user and create corresponding KUBECONFIG, that is the best practice to avoid sharing security configures and performs audit. 

In the following, we are going to leverage kubernete x509 based user authentication method to create user in kubernete cluster. 

* * *


## 1. Generating user specific CSR

There are many tools can be used to generate user csr. I will use [OpenSSL](https://www.openssl.org/) for csr creation. 

* Firstly, we need to generate user RSA key pair in below. 

```
> openssl genrsa -out mbi.pem 2048

output: mbi.pem
```

* Second, use user private key to generate user csr

```
> openssl req -new -key mbi.pem -out mbi.csr -subj "/CN=mbi/O=kubernetes"

output: mbi.csr
```
**The /CN name is the user name in kubernete cluster we will register in the following steps**

## 2. Signing the user CSR via Kubernetes ca

This created CSR shall be passed to kubernetes for signing. There is a default csr controller that handles CertificateSigningRequest. 

```
cat > mbi-csr.yaml <<EOF
apiVersion: certificates.k8s.io/v1beta1
kind: CertificateSigningRequest
metadata:
  name: user-request-mbi
spec:
  groups:
  - system:authenticated
  request: $(cat mbi.csr | base64 | tr -d '\n')
  usages:
  - digital signature
  - key encipherment
  - client auth
EOF
```

```
kubectl create -f mbi-csr.yaml

*certificatesigningrequest.certificates.k8s.io/user-request-mbi created*
```

The kubernetes CSR controller will wait for cluster admin to manually approve submited CSR. 

```
kubectl certificate approve user-request-mbi

*certificatesigningrequest.certificates.k8s.io/user-request-mbi approved*
```

```
> kubectl get csr
NAME               AGE   REQUESTOR          CONDITION
user-request-mbi   58s   kubernetes-admin   Approved,Issued
```

The signed user certificate (crt) is embeded in approved csr metadate. We shall copy it out via below command. 

```
kubectl get csr user-request-mbi -o jsonpath='{.status.certificate}' \
    | base64 --decode > mbi.crt
```

The saved crt will be used to generate user specific KUBECONFIG in below step 4.

## 3. Create clusterrolebinding or rolebinding to grant k8s object CRUD previledge for this user

After approve user CSR in kubernete cluster, a User object is created which user name is /CN in certs. In our sample is mbi. 
We shall link user with a cluster role or namespace role by clusterrolebinding or rolebinding.

```
kubectl create clusterrolebinding mbi-admin --clusterrole=admin --user=mbi
```

## 4. Genereate user specific KUBECONFIG

Now you can create user KUBECONFIG for user to access kubernete apiserver. 

```
kubectl config set-cluster kubernetes --server=https://52.12.99.3:6443 --certificate-authority=cluster-ca.crt --embed-certs=true --kubeconfig=mbi-crt.kubeconfig

```

```
kubectl config set-credentials mbi --client-certificate=mbi.crt --client-key=mbi.pem --embed-certs=true --kubeconfig=mbi-crt.kubeconfig

```

```
kubectl config set-context kubernetes --cluster=kubernetes --user=mbi --kubeconfig=mbi-crt.kubeconfig

```

```
kubectl config use-context kubernetes --kubeconfig=mbi-crt.kubeconfig

```

```
> export KUBECONFIG=./mbi-crt.kubeconfig

> kubectl get po
No resources found.

> kubectl get svc
NAME         TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)   AGE
kubernetes   ClusterIP   10.96.0.1    <none>        443/TCP   5h53m
```
