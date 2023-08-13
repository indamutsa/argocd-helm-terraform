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

Change the folder to terraform-cloud and initialize terraform to run argocd

```bash
terraform init
```

Apply it to run argocd

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

Decode the password | the username is `admin`

```bash
echo -n 'password from above' | base64 -d
```

Port forward the argocd server

```bash
kubectl port-forward svc/argocd-server -n argocd 8080:443
```

and follow the instructions.

After that we can add application yaml to instruct argocd to track our repo

We will add some configuration and build agent to automate the build process.

<!-- prettier-ignore-start -->
❯ tree
.
├── 1-example
│   └── application.yaml
├── app
│   ├── 0-namespace.yaml
│   └── 1-deployment.yaml
├── argocd-values.yaml
├── build-agent.sh
├── README.md
├── terraform-cloud
│   ├── 0-provider.tf
│   ├── 1-provider.tf
│   ├── argocd-default-values.yaml
│   ├── argocd-values.yaml
│   ├── terraform.tfstate
│   ├── terraform.tfstate.backup
│   └── values
│       └── argocd.yaml
└── :wq

10 directories, 21 files

<!-- prettier-ignore-end -->

Create a repo to track with argocd

```bash
gh repo create
```

The first example is made of a simple application that will be deployed in the argo namespace. We have a folder at the root called app which contains the namespace and the deployment yaml files. And we have folder called example 1. This folder contains the application.yaml file. The application.yaml file is the one that will be used by argocd to deploy the application.

First apply the application.yaml file to argocd

```bash
kubectl apply -f 1-example/application.yaml
```

Now change the tag and push the image to the registry using the build-agent.sh script. The build-agent.sh script will build the image and push it to the registry, and push the changes to the repo after updating 1-deployment.yaml file with the new tag using the `sed` command.

```bash
./build-agent.sh v1.0.6
```

We shall see that the application is deployed in the argo namespace inside the dashboard of argocd `http://localhost:8080`.

---

Now let us introduce another type of deployment.
We have have 2-example folder which contains the application.yaml. It is called App of Apps pattern and it is used to manage several applications with a single application.

<!-- prettier-ignore-start -->
❯ tree
.
├── 1-example
│   └── application.yaml
├── 2-example
│   └── application.yaml
├── app
│   ├── 0-namespace.yaml
│   └── 1-deployment.yaml
├── argocd-values.yaml
├── build-agent.sh
├── environments
│   └── staging
│       ├── app
│       │   ├── 0-namespace.yaml
│       │   └── 1-deployment.yaml
│       ├── apps
│       │   ├── app.yaml
│       │   └── second-app.yaml
│       └── second-app
│           ├── 0-namespace.yaml
│           └── 1-deployment.yaml
├── README.md
├── terraform-cloud
│   ├── 0-provider.tf
│   ├── 1-provider.tf
│   ├── argocd-default-values.yaml
│   ├── argocd-values.yaml
│   ├── terraform.tfstate
│   ├── terraform.tfstate.backup
│   └── values
│       └── argocd.yaml
└── :wq

10 directories, 21 files

<!-- prettier-ignore-end -->

The tree structure above introduces a way to use the application in 2-example folder to deploy the applications in the environments folder. The environments folder contains the staging folder which contains the app and second-app folders.

The app and second-app folders contain the namespace and deployment yaml files. The staging folder contains the app.yaml and second-app.yaml files.

The app.yaml and second-app.yaml files are the ones that will be used by argocd to deploy the applications by using the application in 2-example folder.

Apply the application.yaml in 2-example folder to argocd

```bash
kubectl apply -f 2-example/application.yaml
```

Delete the application in example 2 folder to clean up the resources

```bash
kubectl delete -f 2-example/application.yaml
```

---

For private image and github repository:

Create a private image in docker

<!-- ```bash
helm install argocd -n argocd --create-namespace argo/argo-cd --version 3.35.4 -f terraform/argocd-default-values.yaml
```` -->
