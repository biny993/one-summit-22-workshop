
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

Refer to https://github.com/biny993/one-summit-22-workshop/Readme.md

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
rm -rf gitea/
```

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

### history for reference

```sh

    1  exit
    2  sudo ls
    3  exit
    4  ls
    5  export HTTPS_PROXY=http://147.11.252.42:9090
    6  git clone https://github.com/nephio-project/one-summit-22-workshop.git
    7  cd one-summit-22-workshop/nephio-ansible-install
    8  mkdir -p inventory
    9  touch inventory/nephio.yaml
   10  vi inventory/nephio.yaml
   11  scp inventory/nephio.yaml 128.224.115.22:~/nephio/
   12  python3 -m venv .venv
   13  sudo ap install python3.10
   14  sudo apt install python3.10
   15  python -v
   16  python --version
   17  python3 -v
   18  python3 -V
   19  python3 -m venv .venv
   20  sudo apt install python3.10-venv
   21  python3 -m venv .venv
   22  source .venv/bin/activate
   23  pip install --upgrade pip
   24  pip install ansible
   25  pip install pygithub
   26  ansible-galaxy collection install community.general
   27  ansible-galaxy collection install community.docker
   28  ansible-playbook playbooks/install-prereq.yaml
   29  ifconfig
   30  sudo apt install net-tools
   31  ifconfig
   32  ping 10.0.10.4
   33  ssh biny993@10.0.10.4
   34  ls
   35  vi inventory/nephio.yaml
   36  ssh-keygen -t rsa -b 4096
   37  ls ~/.ssh/
   38  file ~/.ssh/id_rsa
   39  ssh-copy-id biny993@10.0.10.4
   40  ssh biny993@10.0.10.4
   41  ansible-playbook playbooks/install-prereq.yaml
   42  history
   43  sudo ls
   44  ssh biny993@10.0.10.4
   45  sudo ls
   46  exit
   47  sudo ls
   48  sudoer
   49  nano /etc/sudoers
   50  ls /etc/sudoers
   51  cat /etc/sudoers
   52  sudo cat /etc/sudoers
   53  sudo nano /etc/sudoers
   54  sudo ls
   55  exit
   56  sudo ls
   57  ls
   58  cd
   59  cd one-summit-22-workshop/
   60  ls
   61  source .venv/bin/activate
   62  cd nephio-ansible-install/
   63  source .venv/bin/activate
   64  ansible-playbook playbooks/install-prereq.yaml
   65  sudo docker ps
   66  sudo docker images
   67  sudo docker images -a
   68  ansible-playbook playbooks/install-prereq.yaml
   69  ping gcr.io
   70  ping www.baidu.com
   71  export HTTPS_PROXY=http://147.11.252.42:9090
   72  ping gcr.io
   73  ls
   74  less inventory/nephio.yaml
   75  sudo docker pull gcr.io/kpt-fn/apply-replacements:v0.1.1
   76  sudo docker images
   77  sudo docker pull ubuntu
   78  sudo docker images
   79  sudo docker images -a
   80  sudo docker pull gcr.io/kpt-fn/apply-replacements:v0.1.1
   81  cat /etc/sysconfig/docker
   82  sudo docker images
   83  sudo docker ps
   84  sudo docker ps -a
   85  sudo docker images
   86  curl
   87  curl localhost:3000
   88  sudo docker ps
   89  sudo docker images
   90  sudo docker ps
   91  sudo docker ps -a
   92  sudo docker stop gitea
   93  sudo docker rm gitea
   94  exit
   95  kubectl get pods
   96  cd one-summit-22-workshop/nephio-ansible-install/
   97  ls
   98  less inventory/nephio.yaml
   99  grep -r 'deploy nephio-system on mgmt cluster' *
  100  less roles/nephio/deploy/tasks/mgmtcluster_files.yaml
  101  ls ~/.kube/mgmt-config
  102  cat ~/.kube/mgmt-config
  103  kubectl -f ~/.kube/mgmt-config get pods
  104  kubectl -f ~/.kube/mgmt-config get hosts
  105  kubectl -k ~/.kube/mgmt-config get hosts
  106  kubectl  -h
  107  kubectl options
  108  kubectl --kubeconfig=~/.kube/mgmt-config get hosts
  109  kubectl --kubeconfig=/home/biny993/.kube/mgmt-config get hosts
  110  kubectl --kubeconfig=/home/biny993/.kube/mgmt-config get host
  111  kubectl --kubeconfig=/home/biny993/.kube/mgmt-config get pods
  112  kubectl --kubeconfig=/home/biny993/.kube/mgmt-config get -a
  113  kubectl --kubeconfig=/home/biny993/.kube/mgmt-config get all
  114  kubectl --kubeconfig=/home/biny993/.kube/mgmt-config get all --all-namespaces
  115  ls ~/kube/
  116  ls ~/.kube/
  117  cat ~/.kube/edge1-config
  118  cat ~/.kube/mgmt-config
  119  clear
  120  grep -r 'Deploy cluster' *
  121  less roles/cluster/deploy/tasks/cluster_files.yaml
  122  kind -h
  123  sudo docker images
  124  sudo docker ps
  125  less roles/cluster/deploy/tasks/cluster_files.yaml
  126  sudo docker ps
  127  sudo docker ps -a
  128  kubectl --kubeconfig=/home/biny993/.kube/mgmt-config get all --all-namespaces
  129  kubectl --kubeconfig=/home/biny993/.kube/mgmt-config -n nephio-webui get pdos
  130  kubectl --kubeconfig=/home/biny993/.kube/mgmt-config -n nephio-webui get pods
  131  kubectl --kubeconfig=/home/biny993/.kube/mgmt-config -n nephio-webui logs -f nephio-webui-fb6bb9479-7pzbb
  132  kubectl --kubeconfig=/home/biny993/.kube/mgmt-config -n nephio-webui get pods
  133  kubectl --kubeconfig=/home/biny993/.kube/mgmt-config -n nephio-webui get svc
  134  less playbooks/configure-nephio.yaml
  135  less playbooks/deploy-clusters.yaml
  136  less playbooks/configure-nephio.yaml
  137  cd playbooks/
  138  ls
  139  cd ..
  140  less playbooks/configure-nephio.yaml
  141  grep -r role *
  142  ls roles/nephio/config/tasks/main.yaml
  143  less roles/nephio/config/tasks/main.yaml
  144  kubectl --kubeconfig=/home/biny993/.kube/mgmt-config -n nephio-webui get pods
  145  grep -r webui *
  146  less roles/nephio/deploy/tasks/mgmtcluster_files.yaml
  147  kubectl --kubeconfig=/home/biny993/.kube/mgmt-config -n nephio-webui get pods
  148  kubectl --kubeconfig=/home/biny993/.kube/mgmt-config -n nephio-webui logs -f nephio-webui-fb6bb9479-7pzbb
  149  exit
  150  ls
  151  cd one-summit-22-workshop/
  152  ls
  153  cd nephio-ansible-install/
  154  ls
  155  vi inventory/nephio.yaml
  156  source .venv/bin/activate
  157  ansible-playbook playbooks/create-repos.yaml
  158  vi inventory/nephio.yaml
  159  ansible-playbook playbooks/create-repos.yaml
  160  ansible-playbook playbooks/deploy-clusters.yaml
  161  ansible-playbook playbooks/configure-nephio.yaml
  162  netstat -tunpl
  163  netstat -tunplba
  164  netstat -tunplv
  165  netstat -tunplve
  166  sudo docker ps
  167  sudo docker logs -f mgmt-control-plane
  168  sudo docker ps
  169  sudo docker logs -f mgmt-control-plane
  170  grep -r webui *
  171  less roles/nephio/deploy/templates/nephio-ingress.j2
  172  less roles/nephio/deploy/tasks/mgmtcluster_files.yaml
  173  exit
  174  systemctl status docker
  175  sudo vim /etc/systemd/system/docker.service.d/http-proxy.conf
  176  sudo vi /etc/systemd/system/docker.service.d/http-proxy.conf
  177  sudo mkdir -p /etc/systemd/system/docker.service.d
  178  sudo vi /etc/systemd/system/docker.service.d/http-proxy.conf
  179  sudo systemctl daemon-reload
  180  sudo systemctl restart docker
  181  history
  182  ls
  183  cd one-summit-22-workshop/nephio-ansible-install/
  184  source .venv/bin/activate
  185  ansible-playbook playbooks/install-prereq.yaml
  186  ansible-playbook playbooks/create-repos.yaml
  187  ansible-playbook playbooks/create-gitea.yaml
  188  ansible-playbook playbooks/create-gitea-repos.yaml
  189  vi inventory/nephio.yaml
  190  ansible-playbook playbooks/create-gitea-repos.yaml
  191  ansible-playbook playbooks/destroy-clusters.yaml
  192  ansible-playbook playbooks/install-prereq.yaml
  193  ansible-playbook playbooks/create-repos.yaml
  194  ls
  195  kubectl --kubeconfig=/home/biny993/.kube/mgmt-config -n nephio-webui get pods
  196  kubectl --kubeconfig=/home/biny993/.kube/mgmt-config -n nephio-webui get cm
  197  kubectl --kubeconfig=/home/biny993/.kube/mgmt-config -n nephio-webui get cm nephio-webui-config
  198  kubectl --kubeconfig=/home/biny993/.kube/mgmt-config -n nephio-webui get cm nephio-webui-config -o yaml
  199  ls
  200  cd one-summit-22-workshop/
  201  ls
  202  cd nephio-ansible-install/
  203  ls
  204  vi inventory/nephio.yaml
  205  source .venv/bin/activate
  206  ansible-playbook playbooks/configure-nephio.yaml
  207  kubectl --kubeconfig=/home/biny993/.kube/mgmt-config -n nephio-webui get cm nephio-webui-config -o yaml
  208  kubectl --kubeconfig=/home/biny993/.kube/mgmt-config -n nephio-webui get cm
  209  kubectl --kubeconfig=/home/biny993/.kube/mgmt-config -n nephio-webui get cm nephio-webui-config -o yaml
  210  kubectl --kubeconfig=/home/biny993/.kube/mgmt-config -n nephio-webui edit cm nephio-webui-config
  211  kubectl --kubeconfig=/home/biny993/.kube/mgmt-config -n nephio-webui get cm nephio-webui-config -o yaml
  212  kubectl --kubeconfig=/home/biny993/.kube/mgmt-config -n nephio-webui get pod
  213  kubectl --kubeconfig=/home/biny993/.kube/mgmt-config -n nephio-webui logs -f nephio-webui-fb6bb9479-7pzbb
  214  kubectl --kubeconfig=/home/biny993/.kube/mgmt-config -n nephio-webui delete pod nephio-webui-fb6bb9479-7pzbb
  215  kubectl --kubeconfig=/home/biny993/.kube/mgmt-config -n nephio-webui logs -f nephio-webui-fb6bb9479-7pzbb
  216  kubectl --kubeconfig=/home/biny993/.kube/mgmt-config -n nephio-webui get pod
  217  kubectl --kubeconfig=/home/biny993/.kube/mgmt-config -n nephio-webui logs -f nephio-webui-fb6bb9479-zhszw
  218  kubectl --kubeconfig=/home/biny993/.kube/mgmt-config -n nephio-webui edit cm nephio-webui-config
  219  kubectl --kubeconfig=/home/biny993/.kube/mgmt-config -n nephio-webui get pod
  220  kubectl --kubeconfig=/home/biny993/.kube/mgmt-config -n nephio-webui delete pod nephio-webui-fb6bb9479-zhszw
  221  kubectl --kubeconfig=/home/biny993/.kube/mgmt-config -n nephio-webui get pod
  222  kubectl --kubeconfig=/home/biny993/.kube/mgmt-config -n nephio-webui logs -f nephio-webui-fb6bb9479-4t9cw
  223  kubectl --kubeconfig=/home/biny993/.kube/mgmt-config -n nephio-webui get cm
  224  kubectl --kubeconfig=/home/biny993/.kube/mgmt-config -n nephio-webui get cm nephio-webui-config
  225  kubectl --kubeconfig=/home/biny993/.kube/mgmt-config -n nephio-webui get cm nephio-webui-config -o yaml
  226  kubectl --kubeconfig=/home/biny993/.kube/mgmt-config -n nephio-webui get cm nephio-webui-config -o yaml |grep localhost
  227  kubectl --kubeconfig=/home/biny993/.kube/mgmt-config -n nephio-webui get cm nephio-webui-config -o yaml
  228  clear
  229  kubectl --kubeconfig=/home/biny993/.kube/mgmt-config -n nephio-webui get all --all-namespaces
  230  kubectl --kubeconfig ~/.kube/mgmt-config -n nephio-system get pods
  231  ansible-playbook playbooks/create-gitea.yaml
  232  ansible-playbook playbooks/destroy-clusters.yaml
  233  kubectl --kubeconfig ~/.kube/mgmt-config -n nephio-system get pods
  234  kind help
  235  kind get
  236  kind get clusters
  237  ls
  238  vi inventory/nephio.yaml
  239  ansible-playbook playbooks/deploy-clusters.yaml
  240  ansible-playbook playbooks/configure-nephio.yaml
  241  cd one-summit-22-workshop/nephio-ansible-install/
  242  source .venv/bin/activate
  243  history
  244  kind get clusters
  245  kubectl --kubeconfig=/home/biny993/.kube/mgmt-config -n nephio-webui get pods
  246  kubectl --kubeconfig=/home/biny993/.kube/mgmt-config -n nephio-webui get cms
  247  kubectl --kubeconfig=/home/biny993/.kube/mgmt-config -n nephio-webui get cm
  248  kubectl --kubeconfig=/home/biny993/.kube/mgmt-config -n nephio-webui get cm nephio-webui-config
  249  kubectl --kubeconfig=/home/biny993/.kube/mgmt-config -n nephio-webui get cm nephio-webui-config -o yaml
  250  ansible-playbook playbooks/destroy-clusters.yaml
  251  ls
  252  ls nind/
  253  ls inventory/
  254  pwd
  255  ls ..
  256  ls ../package
  257  ls ../packages
  258  ls ~/
  259  ls ~/clab-nephio-topology/
  260  ls ~/
  261  ls
  262  ls ~/
  263  ls ~/ -a
  264  ls ~/.ansible/
  265  ls ~/.ansible/tmp/
  266  ls ~/.ansible/cp
  267  ls ~/.ansible/collections/
  268  ls ~/.ansible/collections/ansible_collections/
  269  ls ~/.ansible/collections/ansible_collections/community
  270  ls ~/.ansible/collections/ansible_collections/community/general/
  271  ansible-playbook playbooks/install-prereq.yaml
  272  kind get clusters
  273  ansible-playbook playbooks/deploy-clusters.yaml
  274  ansible-playbook playbooks/configure-nephio.yaml
  275  kubectl --kubeconfig=/home/biny993/.kube/mgmt-config -n nephio-webui get cm nephio-webui-config -o yaml
  276  ls
  277  less ansible.cfg
  278  ls
  279  git status
  280  ls inventory/
  281  ls
  282  ls nind
  283  cd
  284  ls
  285  pwd
  286  ls -a
  287  ls .ansible/
  288  ls .ansible/galaxy_
  289  ls .ansible/galaxy_cache/
  290  ls .ansible/
  291  ls .ansible/tmp
  292  ls .ansible/tmp -a
  293  ls /tmp
  294  ls /tmp/nephio-install/
  295  ls /tmp/nephio-install/nephio-webui/
  296  cd /tmp/nephio-install/nephio-webui/
  297  git config -l
  298  ls
  299  less config-map.yaml
  300  ls
  301  cd ..
  302  rm -rf nephio-webui/
  303  cd -
  304  cd
  305  cd one-summit-22-workshop/nephio-ansible-install/
  306  ansible-playbook playbooks/install-prereq.yaml
  307  cat /tmp/nephio-install/nephio-webui/config-map.yaml
  308  ansible-playbook playbooks/configure-nephio.yaml
  309  kubectl --kubeconfig=/home/biny993/.kube/mgmt-config -n nephio-webui get cm nephio-webui-config -o yaml
  310  ls roles/
  311  ls roles/nephio/
  312  ls roles/nephio/install/
  313  ls roles/nephio/install/tasks/
  314  ls roles/nephio/deploy/
  315  ls roles/nephio/deploy/tasks/
  316  ansible-playbook playbooks/destroy-clusters.yaml
  317  ansible-playbook playbooks/deploy-clusters.yaml
  318  ansible-playbook playbooks/configure-nephio.yaml
  319  kubectl --kubeconfig=/home/biny993/.kube/mgmt-config -n nephio-webui get cm nephio-webui-config -o yaml
  320  kubectl --kubeconfig=/home/biny993/.kube/mgmt-config -n nephio-webui get pods
  321  kubectl --kubeconfig=/home/biny993/.kube/mgmt-config -n nephio-webui logs -f nephio-webui-fb6bb9479-5925b
  322  ansible-playbook playbooks/destroy-clusters.yaml
  323  vi inventory/nephio.yaml
  324  ansible-playbook playbooks/create-gitea.yaml
  325  ansible-playbook playbooks/create-gitea-repos.yaml
  326  ansible-playbook playbooks/create-gitea.yaml
  327  ls gitea
  328  ansible-playbook playbooks/create-gitea.yaml
  329  vi inventory/nephio.yaml
  330  ansible-playbook playbooks/create-gitea.yaml
  331  ansible-playbook playbooks/create-gitea-repos.yaml
  332  cat inventory/nephio.yaml
  333  ifconfig
  334  ANSIBLE_DEBUG=true ANSIBLE_VERBOSITY=4 ansible-playbook playbooks/create-gitea-repos.yaml
  335  vi inventory/nephio.yaml
  336  ANSIBLE_DEBUG=true ANSIBLE_VERBOSITY=4 ansible-playbook playbooks/create-gitea-repos.yaml
  337  ANSIBLE_DEBUG=true ANSIBLE_VERBOSITY=4 ansible-playbook playbooks/create-gitea-repos.yaml |grep -i proxy
  338  ansible-playbook playbooks/create-gitea-repos.yaml
  339  ansible-playbook playbooks/deploy-clusters.yaml
  340  ansible-playbook playbooks/configure-nephio.yaml
  341  kubectl --kubeconfig=/home/biny993/.kube/mgmt-config -n nephio-webui get pods
  342  kubectl --kubeconfig=/home/biny993/.kube/mgmt-config -n nephio-webui logs -f nephio-webui-fb6bb9479-flxj7
  343  sudo docker images
  344  sudo docker ps
  345  sudo docker logs gitea
  346  sudo docker exec gitea  gitea admin user create    --username biny993    --password Li69nux*    --must-change-password=false    --access-token    --email=nephio@nephio.org > /home/biny993/gitea/nephio-create
  347  sudo docker exec -it gitea -- sh
  348  sudo docker exec -it gitea sh
  349  sudo docker exec gitea  gitea admin user create    --username biny993    --password Li69nux*    --must-change-password=false    --access-token    --email=nephio@nephio.org > /home/biny993/gitea/nephio-create
  350  echo $1
  351  echo $?
  352  ls gitea/ssh
  353  ls gitea
  354  ls gitea/nephio-create
  355  less gitea/nephio-create
  356  sudo docker ps
  357  sudo docker stop gitea
  358  sudo docker rm gitea
  359  ls
  360  ls gitea/
  361  ls gitea/nephio-create
  362  rm gitea/nephio-create
  363  ls gitea
  364  less gitea/nephio-create
  365  sudo docker stop gitea
  366  rm gitea/nephio-create
  367  sudo docker rm gitea
  368  rm -rf gitea/
  369  ls
  370  ls gitea/
  371  ls gitea/nephio-gitea-token
  372  cat gitea/nephio-gitea-token
  373  curl http://localhost:3000
  374  curl "http://localhost:3000/api/v1/version"
  375  curl "http://10.0.10.4:3000/api/v1/version"
  376  curl -v "http://localhost:3000/api/v1/version"
  377  cd gitea/
  378  ls
  379  ls ssh
  380  ls
  381  ls git
  382  ls queues/
  383  ls custom/
  384  ls data/
  385  cd ..
  386  ls
  387  ls gitea/
  388  netstat -tunlp
  389  kind get clusters
  390  kubectl --kubeconfig=/home/biny993/.kube/mgmt-config -n nephio-webui logs -f nephio-webui-fb6bb9479-4t9cw
  391  kubectl --kubeconfig=/home/biny993/.kube/mgmt-config -n nephio-webui logs -f nephio-webui-fb6bb9479-flxj7
  392  ls
  393  mkdir nephio-poc1
  394  cd nephio-poc1/
  395  cat<<EOF>topo.yaml
  396  apiVersion: nf.nephio.org/v1alpha1
  397  kind: FiveGCoreTopology
  398  metadata:
  399    name: fivegcoretopology-sample
  400  spec:
  401    upfs:
  402      - name: "agg-layer"
  403        selector:
  404          matchLabels:
  405            nephio.org/region: us-central1
  406            nephio.org/site-type: edge
  407        namespace: "upf"
  408        upf:
  409          upfClassName: "free5gc-upf"
  410          capacity:
  411            uplinkThroughput: "1G"
  412            downlinkThroughput: "10G"
  413          n3:
  414            - networkInstance: "sample-vpc"
  415              networkName: "sample-n3-net"
  416          n4:
  417            - networkInstance: "sample-vpc"
  418              networkName: "sample-n4-net"
  419          n6:
  420            - dnn: "internet"
  421              uePool:
  422                networkInstance: "sample-vpc"
  423                networkName: "ue-net"
  424                prefixSize: "16"
  425              endpoint:
  426                networkInstance: "sample-vpc"
  427                networkName: "sample-n6-net"
  428  EOF
  429  cat topo.yaml
  430  kubectl --kubeconfig ~/.kube/mgmt-config apply -f topo.yaml
  431  kubectl --kubeconfig ~/.kube/mgmt-config -n nephio-system get pods
  432  kubectl --kubeconfig ~/.kube/mgmt-config get ns
  433  kubectl --kubeconfig ~/.kube/mgmt-config get pods --all-namespaces
  434  ls ~/.kube/
  435  kubectl --kubeconfig ~/.kube/region1-config get pods --all-namespaces
  436  kubectl --kubeconfig ~/.kube/edge1-config get pods --all-namespaces
  437  kubectl --kubeconfig ~/.kube/mgmt-config get pods --all-namespaces
  438  kubectl --kubeconfig ~/.kube/mgmt-config -n  nephio-system get pods
  439  kubectl --kubeconfig ~/.kube/mgmt-config -n  nephio-system logs -f package-deployment-controller-controller-8644678db-5kzrk
  440  kubectl --kubeconfig ~/.kube/mgmt-config -n  nephio-system logs -f nf-injector-controller-69667b665f-krdsl
  441  kubectl --kubeconfig ~/.kube/mgmt-config get all --all-namespaces
  442  kubectl --kubeconfig ~/.kube/mgmt-config get resources --all-namespaces
  443  kubectl --kubeconfig ~/.kube/mgmt-config get resources
  444  kubectl --kubeconfig ~/.kube/mgmt-config get resource
  445  kubectl --kubeconfig ~/.kube/mgmt-config get
  446  kubectl --kubeconfig ~/.kube/mgmt-config get resource
  447  kubectl --kubeconfig ~/.kube/mgmt-config get customresourcedefinitions.apiextensions.k8s.io
  448  kubectl --kubeconfig ~/.kube/mgmt-config get customresourcedefinitions.apiextensions.k8s.io nf.nephio.org
  449  kubectl --kubeconfig ~/.kube/mgmt-config get nf.nephio.org
  450  kubectl --kubeconfig ~/.kube/mgmt-config -n  nephio-system logs -f nf-injector-controller-69667b665f-krdsl
  451  kubectl --kubeconfig=/home/biny993/.kube/mgmt-config -n nephio-webui logs -f nephio-webui-fb6bb9479-4t9cw
  452  kubectl --kubeconfig=/home/biny993/.kube/mgmt-config -n nephio-webui logs -f nephio-webui-fb6bb9479-flxj7
  453  kubectl --kubeconfig=/home/biny993/.kube/mgmt-config -n nephio-webui logs -f nephio-webui-fb6bb9479-4t9cw
  454  kubectl --kubeconfig=/home/biny993/.kube/mgmt-config -n nephio-webui exec -it nephio-webui-fb6bb9479-4t9cw bash
  455  kubectl --kubeconfig=/home/biny993/.kube/mgmt-config -n nephio-webui exec -it nephio-webui-fb6bb9479-4t9cw sh
  456  kubectl --kubeconfig=/home/biny993/.kube/mgmt-config -n nephio-webui logs -f nephio-webui-fb6bb9479-4t9cw
  457  kubectl --kubeconfig=/home/biny993/.kube/mgmt-config -n nephio-webui logs -f nephio-webui-fb6bb9479-flxj7 main
  458  kubectl --kubeconfig=/home/biny993/.kube/mgmt-config -n nephio-webui exec -it nephio-webui-fb6bb9479-flxj7 -c main sh
  459  kubectl --kubeconfig=/home/biny993/.kube/mgmt-config -n nephio-webui exec -it nephio-webui-fb6bb9479-flxj7 -c main bash
  460  kubectl --kubeconfig=/home/biny993/.kube/mgmt-config -n nephio-webui logs -f nephio-webui-fb6bb9479-flxj7 main
  461  kubectl --kubeconfig=/home/biny993/.kube/mgmt-config -n nephio-webui logs -f nephio-webui-fb6bb9479-flxj7
  462  kubectl --kubeconfig ~/.kube/edge-1.config -n upf get deploy,cm,network-attachment-definition,upfdeployment,po
  463  kubectl --kubeconfig ~/.kube/edge1.config -n upf get deploy,cm,network-attachment-definition,upfdeployment,po
  464  kubectl --kubeconfig ~/.kube/edge1-config -n upf get deploy,cm,network-attachment-definition,upfdeployment,po
  465  kubectl --kubeconfig ~/.kube/edge1-config get ns
  466  kubectl --kubeconfig ~/.kube/mgmt-config -n nephio-system get pods
  467  kubectl --kubeconfig ~/.kube/mgmt-config get pods --all-namespaces
  468  kubectl --kubeconfig ~/.kube/edge1-config get pods --all-namespaces
  469  kubectl --kubeconfig ~/.kube/mgmt-config get pods nephio-5gc-controller-7964d99b4c-bxhpg
  470  kubectl --kubeconfig ~/.kube/mgmt-config -n nephio-system get pods nephio-5gc-controller-7964d99b4c-bxhpg
  471  kubectl --kubeconfig ~/.kube/mgmt-config -n nephio-system logs -f nephio-5gc-controller-7964d99b4c-bxhpg
  472  ls
  473  cd nephio-poc1/
  474  ls
  475  cat topo.yaml
  476  kubectl --kubeconfig ~/.kube/mgmt-config get nf.nephio.org/v1alpha1
  477  kubectl --kubeconfig ~/.kube/mgmt-config get FiveGCoreTopology
  478  kubectl --kubeconfig ~/.kube/mgmt-config describe FiveGCoreTopology
  479  kubectl --kubeconfig ~/.kube/mgmt-config describe FiveGCoreTopology fivegcoretopology-sample
  480  clear
  481  kubectl --kubeconfig ~/.kube/mgmt-config describe FiveGCoreTopology fivegcoretopology-sample
  482  cd
  483  ls
  484  cd one-summit-22-workshop/nephio-ansible-install/
  485  ls
  486  scp inventory/nephio.yaml 128.224.115.22:~/nephio/
  487  scp inventory/nephio.yaml 128.224.115.22:~/nephio/inventory-nephio-v1.yaml
  488  ls
  489  cd
  490  ls
  491  cd nephio-poc1/
  492  ls
  493  cat topo.yaml
  494  kubectl --kubeconfig ~/.kube/mgmt-config get all --all-namespaces
  495  kubectl --kubeconfig=/home/biny993/.kube/mgmt-config -n nephio-system logs -f deployment.apps/nephio-5gc-controller
  496  kubectl --kubeconfig=/home/biny993/.kube/mgmt-config -n nephio-system get FiveGCoreTopology
  497  kubectl --kubeconfig=/home/biny993/.kube/mgmt-config  get FiveGCoreTopology --all-namespaces
  498  kubectl --kubeconfig=/home/biny993/.kube/mgmt-config -n nephio-system get FiveGCoreTopology
  499  kubectl --kubeconfig ~/.kube/mgmt-config get all --all-namespaces
  500  kubectl --kubeconfig ~/.kube/mgmt-config get FiveGCoreTopology
  501  kubectl --kubeconfig ~/.kube/mgmt-config get FiveGCoreTopology fivegcoretopology-sample
  502  kubectl --kubeconfig ~/.kube/mgmt-config get FiveGCoreTopology fivegcoretopology-sample -o yaml
  503  kubectl --kubeconfig ~/.kube/mgmt-config get FiveGCoreTopology fivegcoretopology-sample
  504  kubectl --kubeconfig ~/.kube/mgmt-config describe FiveGCoreTopology fivegcoretopology-sample
  505  kubectl --kubeconfig ~/.kube/mgmt-config api-resources
  506  kubectl --kubeconfig ~/.kube/mgmt-config api-resources  |grep 5g
  507  kubectl --kubeconfig ~/.kube/mgmt-config api-resources  |grep -i 5g
  508  kubectl --kubeconfig ~/.kube/mgmt-config api-resources  |grep -i topo
  509  kubectl --kubeconfig ~/.kube/mgmt-config api-resources  fivegcoretopologies
  510  kubectl --kubeconfig ~/.kube/mgmt-config api-resources  -h
  511  kubectl --kubeconfig ~/.kube/mgmt-config api-resources  -o wide |grep -i topo
  512  kubectl --kubeconfig ~/.kube/mgmt-config get fivegcoretopologies
  513  kubectl --kubeconfig ~/.kube/mgmt-config get fivegcoretopologies -o wide
  514  ls
  515  cd nephio-poc1/
  516  ls
  517  kubectl --kubeconfig ~/.kube/mgmt-config delete -f topo.yaml
  518  kubectl --kubeconfig=/home/biny993/.kube/mgmt-config -n nephio-system logs -f deployment.apps/nephio-5gc-controller
  519  kubectl --kubeconfig=/home/biny993/.kube/mgmt-config -n nephio-system logs -f deployment.apps/nephio-5gc-controller controller
  520  kind get clusters
  521  kubectl --kubeconfig ~/.kube/mgmt-config -n nephio-system get pods nephio-5gc-controller-7964d99b4c-nf9dr controller
  522  kubectl --kubeconfig ~/.kube/mgmt-config -n nephio-system logs -f nephio-5gc-controller-7964d99b4c-nf9dr controller
  523  kubectl --kubeconfig ~/.kube/mgmt-config -n nephio-system get pods
  524  kubectl --kubeconfig ~/.kube/mgmt-config -n nephio-system describe nephio-5gc-controller-7964d99b4c-nf9dr
  525  kubectl --kubeconfig ~/.kube/mgmt-config -n nephio-system describe pod nephio-5gc-controller-7964d99b4c-nf9dr
  526  exit
  527  kubectl --kubeconfig=/home/biny993/.kube/mgmt-config -n nephio-system logs -f deployment.apps/nephio-5gc-controller
  528  kubectl --kubeconfig=/home/biny993/.kube/mgmt-config -n nephio-system logs -f deployment.apps/nephio-5gc-controller main
  529  kubectl --kubeconfig=/home/biny993/.kube/mgmt-config -n nephio-system logs -f deployment.apps/nephio-5gc-controller controller
  530  kubectl --kubeconfig ~/.kube/mgmt-config get pods
  531  kubectl --kubeconfig ~/.kube/mgmt-config get pods --all-namespaces
  532  kubectl --kubeconfig ~/.kube/mgmt-config logs -f package-deployment-controller-controller-8644678db-5kzrk
  533  kubectl --kubeconfig ~/.kube/mgmt-config -n nephio-system logs -f package-deployment-controller-controller-8644678db-5kzrk
  534  kubectl --kubeconfig ~/.kube/mgmt-config -n nephio-system logs -f package-deployment-controller-controller-8644678db-5kzrk controller
  535  kubectl --kubeconfig ~/.kube/mgmt-config -n nephio-system logs -f package-deployment-controller-controller-8644678db-5kzrk controller
  536  ls
  537  cd one-summit-22-workshop/nephio-ansible-install/
  538  ls
  539  source .venv/bin/activate
  540  ls
  541  ansible-playbook playbooks/destroy-clusters.yaml
  542  kind get clusters
  543  ansible-playbook playbooks/create-gitea-repos.yaml
  544  ansible-playbook playbooks/deploy-clusters.yaml
  545  exit
  546  ls
  547  cd one-summit-22-workshop/nephio-ansible-install/
  548  source .venv/bin/activate
  549  ansible-playbook playbooks/configure-nephio.yaml
  550  ls
  551  cd roles/
  552  cd ..
  553  ls /tmp/nephio-install/
  554  history
  555  ansible-playbook playbooks/deploy-clusters.yaml
  556  kubectl --kubeconfig ~/.kube/mgmt-config -n nephio-system logs -f package-deployment-controller-controller-8644678db-fn58n controller
  557  kind get clusters
  558  kind help
  559  ls
  560  cat inventory/nephio.yaml
  561  ls /usr/lib/systemd/system/docker.service
  562  cat /usr/lib/systemd/system/docker.service
  563  cat /usr/lib/systemd/system/docker.socket
  564  grep proxy /usr/lib/systemd/
  565  grep -ir proxy /usr/lib/systemd/
  566  ls /etc/systemd/system/docker.service.d
  567  ls /etc/systemd/system/docker.service.d/http-proxy.conf
  568  cat /etc/systemd/system/docker.service.d/http-proxy.conf
  569  kind get clusters
  570  kubectl --kubeconfig ~/.kube/mgmt-config -n nephio-system logs -f package-deployment-controller-controller-8644678db-fn58n controller
  571  kubectl --kubeconfig ~/.kube/mgmt-config -n nephio-system exec -it  package-deployment-controller-controller-8644678db-fn58n -c controller bash
  572  kubectl --kubeconfig ~/.kube/mgmt-config -n nephio-system exec -it  package-deployment-controller-controller-8644678db-fn58n -c controller sh
  573  kubectl --kubeconfig ~/.kube/mgmt-config -n nephio-system exec -it  package-deployment-controller-controller-8644678db-fn58n -c controller /bin/bash
  574  kubectl --kubeconfig ~/.kube/mgmt-config -n nephio-system exec -it  package-deployment-controller-controller-8644678db-fn58n -c controller /bin/sh
  575  kubectl --kubeconfig ~/.kube/mgmt-config -n nephio-system exec -it  package-deployment-controller-controller-8644678db-fn58n -c controller ls
  576  kubectl --kubeconfig ~/.kube/mgmt-config -n nephio-system exec -it  package-deployment-controller-controller-8644678db-fn58n -c controller ps
  577  kubectl --kubeconfig ~/.kube/mgmt-config -n nephio-system exec -it  package-deployment-controller-controller-8644678db-fn58n -c controller -- ps
  578  kubectl --kubeconfig ~/.kube/mgmt-config -n nephio-system exec -it  package-deployment-controller-controller-8644678db-fn58n -c controller -- /bin/ls
  579  kubectl --kubeconfig ~/.kube/mgmt-config -n nephio-system exec -it  package-deployment-controller-controller-8644678db-fn58n -c controller -- /sbin/ls
  580  kubectl --kubeconfig ~/.kube/mgmt-config -n nephio-system exec -it  package-deployment-controller-controller-8644678db-fn58n -c controller -- bash
  581  kind load
  582  kind load docker-images
  583  kind load docker-image
  584  kind load docker-image -h
  585  kind get clusters
  586  kind get nodes
  587  kind get nodes -h
  588  kind -n mgmt get nodes
  589  kubectl --kubeconfig ~/.kube/mgmt-config -n nephio-system logs -f package-deployment-controller-controller-8644678db-fn58n controller
  590  git status
  591  grep -r https://github.com/biny993/free5gc-packages.git *
  592  cd ..
  593  grep -r https://github.com/nephio-project/free5gc-packages.git *
  594  vi nephio-ansible-install/roles/nephio/config/templates/free-5gc-package.j2
  595  vi packages/participant/repo-free5gc-packages.yaml
  596  cd -
  597  ansible-playbook playbooks/destroy-clusters.yaml
  598  ansible-playbook playbooks/create-gitea-repos.yaml
  599  ansible-playbook playbooks/deploy-clusters.yaml
  600  kubectl --kubeconfig ~/.kube/mgmt-config get pods --all-namespaces
  601  kubectl --kubeconfig=/home/biny993/.kube/mgmt-config -n nephio-webui logs -f nephio-webui-fb6bb9479-flxj7
  602  kubectl --kubeconfig=/home/biny993/.kube/mgmt-config -n nephio-webui get pods
  603  kubectl --kubeconfig=/home/biny993/.kube/mgmt-config get svc --all-namespaces
  604  kubectl --kubeconfig=/home/biny993/.kube/mgmt-config -n nephio-webui get pods
  605  kubectl --kubeconfig=/home/biny993/.kube/mgmt-config get pods --all-namespaces
  606  kubectl --kubeconfig=/home/biny993/.kube/mgmt-config -n nephio-webui get pods
  607  kubectl --kubeconfig=/home/biny993/.kube/mgmt-config -n nephio-webui describe pod nephio-webui-fb6bb9479-97zps
  608  kubectl --kubeconfig=/home/biny993/.kube/mgmt-config -n nephio-webui get pods
  609  kubectl --kubeconfig=/home/biny993/.kube/mgmt-config -n nephio-webui describe pod nephio-webui-fb6bb9479-97zps
  610  kubectl --kubeconfig=/home/biny993/.kube/mgmt-config get pods --all-namespaces
  611  kubectl --kubeconfig=/home/biny993/.kube/mgmt-config -n nephio-webui get pods
  612  kubectl --kubeconfig=/home/biny993/.kube/mgmt-config -n nephio-webui describe pod nephio-webui-fb6bb9479-97zps
  613  kubectl --kubeconfig ~/.kube/edge1-config get pods --all-namespaces
  614  kubectl --kubeconfig ~/.kube/edge2-config get pods --all-namespaces
  615  kubectl --kubeconfig ~/.kube/mgmt-config get fivegcoretopologies -o wide
  616  cd nephio-poc1/
  617  ls
  618  kubectl --kubeconfig=/home/biny993/.kube/mgmt-config -n nephio-system logs -f deployment.apps/nephio-5gc-controller controller
  619  kubectl --kubeconfig ~/.kube/mgmt-config -n nephio-system logs -f package-deployment-controller-controller-8644678db-5kzrk controller
  620  kubectl --kubeconfig ~/.kube/mgmt-config -n nephio-system logs -f package-deployment-controller-controller-8644678db-fn58n controller
  621  ls
  622  kubectl --kubeconfig ~/.kube/mgmt-config delete -f topo.yaml
  623  kubectl --kubeconfig ~/.kube/mgmt-config get fivegcoretopologies -o wide
  624  kubectl --kubeconfig ~/.kube/mgmt-config get api-resource
  625  kubectl --kubeconfig ~/.kube/mgmt-config api-resources
  626  kubectl --kubeconfig ~/.kube/mgmt-config api-resources  |grep package
  627  kubectl --kubeconfig ~/.kube/mgmt-config get packagedeployments
  628  kubectl --kubeconfig ~/.kube/mgmt-config -n nephio-system logs -f package-deployment-controller-controller-8644678db-fn58n controller
  629  kubectl --kubeconfig ~/.kube/mgmt-config apply -f topo.yaml
  630  kubectl --kubeconfig ~/.kube/mgmt-config -n nephio-system logs -f package-deployment-controller-controller-8644678db-fn58n controller
  631  sudo docker ps
  632  sudo docker exec -it mgmt-control-plane sh
  633  sudo docker exec -it mgmt-control-plane bash
  634  sudo docker images
  635  git pull gcr.io/jbelamaric-public/nephio-upf-ipam-fn:latest
  636  docker pull gcr.io/jbelamaric-public/nephio-upf-ipam-fn:latest
  637  sudo docker images
  638  kind load docker-image gcr.io/jbelamaric-public/nephio-upf-ipam-fn:latest --name mgmt
  639  kind --help
  640  kind get nodes
  641  kind get nodes --name mgmt
  642  kind load
  643  kind
  644  kubectl --kubeconfig ~/.kube/mgmt-config -n nephio-system logs -f package-deployment-controller-controller-8644678db-fn58n controller
  645  kubectl --kubeconfig ~/.kube/mgmt-config delete -f topo.yaml
  646  kubectl --kubeconfig ~/.kube/mgmt-config get fivegcoretopologies -o wide
  647  kubectl --kubeconfig ~/.kube/mgmt-config apply -f topo.yaml
  648  kind load docker-image gcr.io/jbelamaric-public/nephio-upf-ipam-fn:latest --name mgmt --nodes mgmt-control-plane
  649  kubectl --kubeconfig ~/.kube/mgmt-config delete -f topo.yaml
  650  kubectl --kubeconfig ~/.kube/mgmt-config apply -f topo.yaml
  651  sudo docker images
  652  sudo docker login hub.docker.io
  653  sudo docker login docker.io
  654  sudo docker images
  655  docker images
  656  docker tags gcr.io/jbelamaric-public/nephio-upf-ipam-fn:latest biny993/nephio-upf-ipam-fn:v20230210
  657  docker tag gcr.io/jbelamaric-public/nephio-upf-ipam-fn:latest biny993/nephio-upf-ipam-fn:v20230210
  658  docker images
  659  docker push biny993/nephio-upf-ipam-fn:v20230210
  660  docker pull gcr.io/jbelamaric-public/nad-inject-fn:latest
  661  docker tag gcr.io/jbelamaric-public/nad-inject-fn:latest biny993/nad-inject-fn:v20230210
  662  docker push biny993/nad-inject-fn:v20230210
  663  kubectl --kubeconfig ~/.kube/mgmt-config delete -f topo.yaml
  664  kind get clusters
  665  kubectl --kubeconfig ~/.kube/mgmt-config get all --namespace
  666  kubectl --kubeconfig ~/.kube/mgmt-config get all --all-namespaces
  667  kubectl --kubeconfig ~/.kube/mgmt-config get pods --all-namespaces
  668  kubectl --kubeconfig=/home/biny993/.kube/mgmt-config -n nephio-webui get pods
  669  kubectl --kubeconfig=/home/biny993/.kube/mgmt-config -n nephio-webui logs -f nephio-webui-fb6bb9479-42fx7
  670  kubectl --kubeconfig ~/.kube/mgmt-config get pods --all-namespaces
  671  kubectl --kubeconfig ~/.kube/mgmt-config -n nephio-system logs -f package-deployment-controller-controller-8644678db-2tblc controller
  672  kubectl --kubeconfig=/home/biny993/.kube/mgmt-config -n nephio-webui logs -f nephio-webui-fb6bb9479-42fx7
  673  cd one-summit-22-workshop/nephio-ansible-install/
  674  ls
  675  ls ~/gitea/
  676  ls ~/gitea/nephio-gitea-token
  677  cat ~/gitea/nephio-gitea-token
  678  exit
  679  ls
  680  cat one-summit-22-workshop/nephio-ansible-install/inventory/nephio.yaml
  681  ls
  682  cd nephio-poc1/
  683  ls
  684  git clone http://localhost:3000/nephio/free5gc-packages.git
  685  ls
  686  cd free5gc-packages/
  687  git remote -l
  688  git remote add https://github.com/biny993/free5gc-packages.git
  689  git remote add github https://github.com/biny993/free5gc-packages.git
  690  git pull github main
  691  export HTTPS_PROXY=http://147.11.252.42:9090
  692  git pull github main
  693  git status
  694  git log
  695  ls
  696  git push origin main
  697  exit
  698  ls
  699  cd nephio-poc1/free5gc-packages/
  700  ls
  701  git tag
  702  git tag free5gc-operator/v2
  703  git tag free5gc-upf/v1
  704  git push origin main
  705  git tag -h
  706  git tags
  707  git tag
  708  git -h
  709  git help
  710  git push -h
  711  git push --tags origin main
  712  git log
  713  git tag
  714  git push --tags github main
  715  export HTTPS_PROXY=http://147.11.252.42:9090
  716  git push --tags github main
  717  sudo docker ps
  718  sudo docker ps -a
  719  git push --tags origin main
  720  ls ~/gitea/
  721  cat ~/gitea/nephio-gitea-token
  722  sudo docker ps -a
  723  kubectl --kubeconfig ~/.kube/mgmt-config get pods --all-namespaces
  724  kubectl --kubeconfig=/home/biny993/.kube/mgmt-config -n nephio-webui describe pod nephio-webui-fb6bb9479-97zps
  725  kubectl --kubeconfig ~/.kube/mgmt-config -n nephio-system describe pod nephio-5gc-controller-7964d99b4c-nf9dr
  726  kubectl --kubeconfig ~/.kube/mgmt-config -n nephio-system describe pod nephio-5gc-controller-7964d99b4c-z62c7
  727  kubectl --kubeconfig ~/.kube/mgmt-config get pods --all-namespaces
  728  cd ..
  729  git clone https://github.com/nephio-project/nephio-packages.git
  730  cd nephio-packages/
  731  ls
  732  git log
  733  git tag
  734  git remote add github https://github.com/biny993/nephio-packages.git
  735  git push --tags github main
  736  git pull github main
  737  git log
  738  git show
  739  git push --tags github main
  740  git add remote gitea http://128.224.115.22:3000/nephio/nephio-packages.git
  741  git remote add gitea http://128.224.115.22:3000/nephio/nephio-packages.git
  742  git push --tags gitea main
  743  git push --tags github main
  744  cat ~/gitea/nephio-gitea-token
  745  cat ~/one-summit-22-workshop/nephio-ansible-install/inventory/nephio.yaml

```

