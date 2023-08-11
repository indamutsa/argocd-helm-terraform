Let ui start by installing ArgoCD from helm chart repository and update the index

#### Create a kind cluster

```bash
cd ArgoCD
kind create cluster --name argocd
```

Next up, I will be running a small container where I will be doing all the work from:
You can skip this part if you already have `kubectl` and `helm` on your machine.

#### Run a working container

```bash
docker run -it --rm --net host -v ${HOME}/.kube/:/root/.kube/ -v ${PWD}:/work -w /work alpine sh
```

#### Install `kubectl`

```bash
apk add --no-cache curl
curl -LO https://storage.googleapis.com/kubernetes-release/release/`curl -s https://storage.googleapis.com/kubernetes-release/release/stable.txt`/bin/linux/amd64/kubectl
chmod +x ./kubectl
mv ./kubectl /usr/local/bin/kubectl
```

#### Install `helm`

```bash
curl -LO https://get.helm.sh/helm-v3.7.2-linux-amd64.tar.gz
tar -C /tmp/ -zxvf helm-v3.7.2-linux-amd64.tar.gz
rm helm-v3.7.2-linux-amd64.tar.gz
mv /tmp/linux-amd64/helm /usr/local/bin/helm
chmod +x /usr/local/bin/helm
```

Now we have `helm` and `kubectl` and can access our `kind` cluster:

#### Check if we can access the cluster

````bash

```bash
kubectl get nodes
NAME                  STATUS   ROLES                  AGE   VERSION
vault-control-plane   Ready    control-plane,master   37s   v1.21.1
````

#### Terraform CLI

```bash
# Get Terraform

curl -o /tmp/terraform.zip -LO https://releases.hashicorp.com/terraform/1.5.5/terraform_1.5.5_linux_amd64.zip
unzip /tmp/terraform.zip
chmod +x terraform && mv terraform /usr/local/bin/
terraform
```

Get ArgoCD helm chart repository

```bash
helm repo add argo https://argoproj.github.io/argo-helm
helm repo update
```

Look for it in the helm repo

```bash
helm search repo argocd
```

To overwrite the default values, we can get the values file from the repo and edit it

```bash
helm show values argo/argo-cd --version 3.35.4 > argocd-values.yaml
```

Initialize terraform

```bash
terraform init
```

Apply it

```bash
terraform apply --auto-approve
```

verify that argocd is running

```bash
kubectl get po -n argocd
```

Get its Secrets:

```bash
kubectl get secret -n argocd
```

Get the initial password

```bash
kubectl get secrets argocd-initial-admin-secret -o yaml -n argocd
```

Decode the password

```bash
echo -n 'password from above' | base64 -d
```

Port forward the argocd server

```bash
kubectl port-forward svc/argocd-server -n argocd 8080:443
```

````
```bash
helm install argocd -n argocd --create-namespace argo/argo-cd --version 3.35.4 -f terraform/argocd-default-values.yaml
````
