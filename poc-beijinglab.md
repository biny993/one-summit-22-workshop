
# Nephio ONES poc exercise in Beijing Lab

## Prerequisites

### create github personal access token (classic)

### workaround network access issue to docker registry of gcr.io

1, cache docker kpt function images by hub.docker.io

- https://hub.docker.com/repository/docker/biny993/nephio-upf-ipam-fn
- https://hub.docker.com/repository/docker/biny993/nad-inject-fn

```sh
docker pull gcr.io/jbelamaric-public/nephio-upf-ipam-fn:latest
sudo docker login docker.io
docker tag gcr.io/jbelamaric-public/nephio-upf-ipam-fn:latest biny993/nephio-upf-ipam-fn:v20230210
docker images
docker push biny993/nephio-upf-ipam-fn:v20230210
docker pull gcr.io/jbelamaric-public/nad-inject-fn:latest
docker tag gcr.io/jbelamaric-public/nad-inject-fn:latest biny993/nad-inject-fn:v20230210
docker push biny993/nad-inject-fn:v20230210
```

2, fork https://github.com/nephio-project/free5gc-packages to https://github.com/biny993/free5gc-packages

```sh
git clone https://github.com/nephio-project/free5gc-packages.git
cd free5gc-packages
git remote add github https://github/biny993/free5gc-packages.git
git push --tags github main
cd -
```
Note: `git push --tags` is needed to sync up tags

3, apply following patch:
- https://github.com/nephio-project/free5gc-packages/commit/7f3b944ad744b3d980a0717466aff7126db569ea

### workaround webui issue

1, fork https://github.com/nephio-project/nephio-packages to https://github.com/biny993/nephio-packages


```sh
git clone https://github.com/nephio-project/nephio-packages.git
cd nephio-packages
git remote add github https://github/biny993/nephio-packages.git
git push --tags github main
cd -
```
Note: `git push --tags` is needed to sync up tags

2, apply following patches:
- https://github.com/nephio-project/nephio-packages/commit/8e8f53d3e701cec02f7c2e270d8db7ab6e63440d
- https://github.com/nephio-project/nephio-packages/commit/766278082d549fa9b8317677a33781b3f5e1823c


## Setup nephio VM over vbox

### VM resource:

    ubuntu 22.04LTS -> this is tested right now
    32G RAM, 8 vcpu -> we can change this based on the amount of kind clusters we need
    50GB disk (default 10GB disk on GCE is too small, 50GB is tested)
    SSH access with a SSH key is setup + username

### VM network:
    NAT Network: nephio1nat

### port forward on NAT Network: nephio1nat

virtualbox -> preference -> networks -> nat natwork -> nephio1nat -> port forward:
    nephio1-ssh 128.224.115.22 6022 10.0.10.4 22
    webui 128.224.115.22 7007 10.0.10.4 7007
    gitea 128.224.115.22 3000 10.0.10.4 3000

### user/pass
    biny993/<password>

### ssh to nephio vm:
ssh -p 6022 biny993@128.224.115.22

## Install Nephio Over Ubuntu20.04 LTS host

### enable password-less login and sudo without password to localhost

```sh
sudo apt update
sudo apt install net-tools
ssh-keygen -t rsa -b 4096
ssh-copy-id biny993@localhost
sudo nano /etc/sudoers
    biny993 ALL=NOPASSWD: ALL

```

### ansible to install nephio poc

```sh
export HTTPS_PROXY=http://147.11.252.42:9090
git clone https://github.com/biny993/one-summit-22-workshop.git

cd one-summit-22-workshop/nephio-ansible-install
mkdir -p inventory
touch inventory/nephio.yaml

cat<<EOF>inventory/nephio.yaml
all:
  vars:
    cloud_user: biny993
    github_username: <github username>
    github_token: <github access token

    gitea_username: nephio
    gitea_password: nephio

    dockerhub_username: <docker username>
    dockerhub_token: <docker access token>
    proxy:
      http_proxy: http://147.11.252.42:9090
      https_proxy: http://147.11.252.42:9090
      no_proxy: 10.*,127.0.0.1,localhost,gitea
    host_os: "linux"  # use "darwin" for MacOS X, "windows" for Windows
    host_arch: "amd64"  # other possible values: "386","arm64","arm","ppc64le","s390x"
    tmp_directory: "/tmp"
    bin_directory: "/usr/local/bin"
    kubectl_version: "1.25.0"
    kubectl_checksum_binary: "sha512:fac91d79079672954b9ae9f80b9845fbf373e1c4d3663a84cc1538f89bf70cb85faee1bcd01b6263449f4a2995e7117e1c85ed8e5f137732650e8635b4ecee09"
    kind_version: "0.17.0"
    cni_version: "0.8.6"
    kpt_version: "1.0.0-beta.23"
    multus_cni_version: "3.9.2"
    nephio:
      install_dir: nephio-install
      packages_url: https://github.com/biny993/nephio-packages.git
    clusters:
      mgmt: {mgmt_subnet: 172.88.0.0/16, pod_subnet: 10.196.0.0/16, svc_subnet: 10.96.0.0/16}
      edge1: {mgmt_subnet: 172.89.0.0/16, pod_subnet: 10.197.0.0/16, svc_subnet: 10.97.0.0/16}
      edge2: {mgmt_subnet: 172.90.0.0/16, pod_subnet: 10.198.0.0/16, svc_subnet: 10.98.0.0/16}
      region1: {mgmt_subnet: 172.91.0.0/16, pod_subnet: 10.199.0.0/16, svc_subnet: 10.99.0.0/16}
  children:
    vm:
      hosts:
        localhost
EOF


python3 -m venv .venv
source .venv/bin/activate
pip install --upgrade pip
pip install ansible
pip install pygithub
ansible-galaxy collection install community.general

ansible-galaxy collection install community.docker

rm -rf /tmp/*
ansible-playbook playbooks/install-prereq.yaml

```

### provision proxy for docker.io

in case install-prereq fails due to networking issue, adding proxy for docker and re-run the install-prereq again

```sh

systemctl status docker
sudo ls /etc/systemd/system/docker.service.d/http-proxy.conf

sudo mkdir -p /etc/systemd/system/docker.service.d
cat<<EOF>~/http-proxy.conf
[Service]
Environment="HTTP_PROXY=http://147.11.252.42:9090"
Environment="HTTPS_PROXY=http://147.11.252.42:9090"
Environment="NO_PROXY=localhost,127.0.0.1,docker-registry.example.com,.wrs.com"
EOF
sudo cp ~/http-proxy.conf /etc/systemd/system/docker.service.d/http-proxy.conf

sudo systemctl daemon-reload
sudo systemctl restart docker

ansible-playbook playbooks/install-prereq.yaml

```

### deploy local gitea repos

```sh
ansible-playbook playbooks/create-gitea.yaml
ansible-playbook playbooks/create-gitea-repos.yaml
```

### migrate free5gc and nephio-package

migrate following github projects repos:


migration of github repo can be done with gitea,
e.g., http://128.224.115.22:3000/repo/migrate?service_type=2


```
https://github.com/biny993/nephio-packages.git

https://github.com/biny993/free5gc-packages.git

```

as local gitea repos:

```
http://gitea:3000/nephio/nephio-packages

http://gitea:3000/nephio/free5gc-packages

```



### deploy cluster
```sh

ansible-playbook playbooks/deploy-clusters.yaml
ansible-playbook playbooks/configure-nephio.yaml

```

### access webui

http://128.224.115.22:7007

### access to gitea

http://128.224.115.22:3000
nephio/nephio

## Deploy free5g-operator and FiveGCoreTopology

Refer to https://github.com/biny993/one-summit-22-workshop#deploy-a-fivegcoretopology

e.g.,

```sh
kubectl --kubeconfig ~/.kube/mgmt-config apply -f /tmp/5gc-topology.yaml
```

## appendix

### Speed up nephio webui access
the nephio webui access latency increases due to low access to github from beijing lab.
replace the github repositories by replace github repos with local ones, e.g., repo over gitea
which is hosted over nephio VM

1, create emtpy gitea repos of following over gitea: http://128.224.115.22:3000

 - http://128.224.115.22:3000/nephio/nephio-packages.git
 - http://128.224.115.22:3000/nephio/free5gc-packages.git

2, clone repos from github and push to gitea repos

```sh
git clone https://github/biny993/free5gc-packages.git
cd free5gc-packages
git remote add gitea http://128.224.115.22:3000/nephio/free5gc-packages.git
git push --tags gitea main
cd -

git clone https://github/biny993/nephio-packages.git
cd nephio-packages
git remote add gitea http://128.224.115.22:3000/nephio/nephio-packages.git
git push --tags gitea main
cd -

```

3, unregister github repos from nephio webui: http://128.224.115.22:7007/config-as-data

- http://128.224.115.22:7007/config-as-data/repositories/nephio-packages
Click "Advanced" tab, click "UNREGISTER REPOSITORY"

- http://128.224.115.22:7007/config-as-data/repositories/free-5-gc-packages
Click "Advanced" tab, click "UNREGISTER REPOSITORY"

4, register gitea repos: http://128.224.115.22:7007/config-as-data

Click "Repositories" tab, click "REGISTER REPOSITORY", fill form with following info:
    URL: http://128.224.115.22:3000/nephio/nephio-packages.git
    Type: Git
    Personal Access Token: <get the token by `cat ~/gitea/nephio-gitea-token` over nephio VM>
    Repository Content: External Blueprints
    Confirm to "REGISTER REPOSITORY"

Click "Repositories" tab, click "REGISTER REPOSITORY", fill form with following info:
    URL: http://128.224.115.22:3000/nephio/free5gc-packages.git
    Type: Git
    Personal Access Token: <get the token by `cat ~/gitea/nephio-gitea-token` over nephio VM>
    Repository Content: External Blueprints
    Confirm to "REGISTER REPOSITORY"


### Uninstall cluster

```sh
ansible-playbook playbooks/destroy-clusters.yaml
```
### destroy gitea

```sh
sudo docker stop gitea
sudo docker rm gitea
rm -rf ~/gitea/
```

### Remove artifacts for re-running install-prereq.yaml

/tmp/cni should be removed

```sh
rm -rf /tmp/cni
```

containerlab must be removed if you want to rerun

~~~sh
sudo rm /usr/bin/containerlab
~~~

### check webui logs

```sh

kubectl --kubeconfig=/home/biny993/.kube/mgmt-config get all --all-namespaces
kubectl --kubeconfig=/home/biny993/.kube/mgmt-config -n nephio-webui get pods
kubectl --kubeconfig=/home/biny993/.kube/mgmt-config -n nephio-webui logs -f nephio-webui-fb6bb9479-7pzbb

```

### check package-deployment controller logs

```sh

kubectl --kubeconfig ~/.kube/mgmt-config -n  nephio-system get pods
kubectl --kubeconfig ~/.kube/mgmt-config -n  nephio-system logs -f package-deployment-controller-controller-8644678db-5kzrk
  
```
### check pods over edges cluster
```sh
kubectl --kubeconfig ~/.kube/edge1-config get pods --all-namespaces
kubectl --kubeconfig ~/.kube/edge2-config get pods --all-namespaces
kubectl --kubeconfig ~/.kube/region1-config get pods --all-namespaces

```
