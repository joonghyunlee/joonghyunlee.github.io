---
layout: post
cover: 'assets/images/cover3.jpg'
navigation: True
title: Install OpenStack via Kolla-ansible
date: 2020-11-02 07:00:00
tags: openstack kolla kolla-ansible
subclass: 'post tag-test tag-content'
logo: 'assets/images/ghost.png'
disqus: true
categories: openstack
---


이 문서에서는 kolla-ansible을 통해 OpenStack을 설치하는 것을 다룬다.

## 사전 준비

### VirtualBox 설치

이 문서에서는 물리 노드 위에 Virtualbox를 이용해 가상 머신 여러 대를 구성하고 이 위에 kolla-ansible로 설치할 것이다. 이를 위해 먼저 Virtualbox를 설치하도록 하자. 혹시 이미 사용할 장비에 Virtualbox가 설치되어 있다면 건너 뛰어도 좋다. Virtualbox를 설치하기 위해 [공식 홈페이지](https://www.virtualbox.org/wiki/Downloads)로부터 자신의 컴퓨터에 맞는 패키지를 찾아 내려받자. 여기서 사용할 장비는 Linux (CentOS 7)이므로 패키지를 다운 받은 후 다음과 같이 설치한다.

```bash
wget https://download.virtualbox.org/virtualbox/6.1.16/VirtualBox-6.1-6.1.16_140961_el7-1.x86_64.rpm
wget https://download.virtualbox.org/virtualbox/6.1.16/Oracle_VM_VirtualBox_Extension_Pack-6.1.16.vbox-extpack

sudo yum install VirtualBox-6.1-6.1.16_140961_el7-1.x86_64.rpm
sudo vboxmanage extpack install Oracle_VM_VirtualBox_Extension_Pack-6.1.16.vbox-extpack
```

### Vagrant 설치

다음으로는 Vagrant를 설치한다. Vagrant 공식 홈페이지로부터 최신 패키지를 내려받은 후 다음과 같이 설치하자.

```bash
wget https://releases.hashicorp.com/vagrant/2.2.10/vagrant_2.2.10_x86_64.rpm

sudo yum install vagrant_2.2.10_x86_64.rpm
```

그리고 필요한 Plugin 들도 다음과 같이 설치해준다.

```bash
vagrant plugin install vagrant-hostmanager
vagrant plugin install vagrant-vbguest
```

### OpenStack Kolla/Kolla-ansible 설치

다음으로 Kolla와 Kolla-ansible 코드를 내려받자. 이 문서에서는 Queens 버전으로 진행할 것이기 때문에 최신 Queens stable 버전으로 체크아웃 한다.

```bash
$ git clone https://opendev.org/openstack/kolla-cli
$ cd kolla/
$ git checkout -t remotes/origin/stable/queens
$ cd ..
$ git clone https://opendev.org/openstack/kolla-ansible
$ cd kolla-ansible
$ git checkout -t remotes/origin/stable/queens
```

## 실행 준비하기

### Vagrant 설정 변경

앞 서 설명한 모든 준비가 끝났다면 다음 경로로 이동하자.

```bash
cd kolla-ansible/contrib/dev/vagrant
```

이동한 폴더에는 kolla/kolla-ansible을 통해 OpenStack 설치 과정을 진행할 수 있도록 준비된 파일들이 있다. 공식 문서대로라면 이 폴더에서 `vagrant up`을 통해 즉시 실행할 수 있어야 하지만, 이 문서에서는 기본 설정을 조금 변경해서 진행할 것이다. 우선  `Vagrantfile.custom` 파일을 생성 해보자. 

```bash
$ cp Vagrantfile.custom.example Vagrantfile.custom
```

이 파일은 사용자 지정이 가능한 설정을 변경할 수 있도록 도와 준다. 이 문서에서는 Virtualbox/Ubuntu 기반의 여러 가상 머신을 구성할 것이므로 다음과 같이 변경한다.

```bash
# Either libvirt or virtualbox
PROVIDER = "virtualbox"

# Either centos or ubuntu
DISTRO = "ubuntu"

# Whether the host network adapter is Wi-Fi.
# On VirtualBox, the user must first manually create a NAT-Network
# named OSNetwork. The default network CIDR must be changed.
# The Neutron external interface will be connected to this Network.
WIFI = true

# Whether to do Multi-node or All-in-One deployment
MULTINODE = true

# The following is only used when deploying in Multi-nodes
NUMBER_OF_CONTROL_NODES = 2
NUMBER_OF_COMPUTE_NODES = 2
NUMBER_OF_STORAGE_NODES = 1
NUMBER_OF_NETWORK_NODES = 1
NUMBER_OF_MONITOR_NODES = 1

NODE_SETTINGS = {
  aio: {
    cpus: 4,
    memory: 4096
  },
  operator: {
    cpus: 2,
    memory: 4096
  },
  control: {
    cpus: 4,
    memory: 8192
  },
  compute: {
    cpus: 2,
    memory: 4096
  },
  storage: {
    cpus: 1,
    memory: 1024
  },
  network: {
    cpus: 2,
    memory: 2048
  },
  monitor: {
    cpus: 1,
    memory: 1024
  }
}
```

위 설정은 멀티 노드 구성을 활성화하고 앞으로의 원활한 테스트를 위해 일부 설정을 조정한 것이다. 각 노드 타입별 갯수와 CPU/RAM 구성은 각자 환경에 맞게 조정해도 좋다. 위 설정 값은 사용한 장비에서 몇 번 테스트하여 얻어진 것이다. 경험 상 Control 장비에 OpenStack 주요 서비스들과 mariadb, rabbitmq 등과 같은 보조 서비스들이 대부분 구동되므로 RAM 요구량이 상대적으로 높다.

위 설정에서 WIFI를 쓰지도 않는데 설정해서 의아하게 생각할 지 모르겠다. 그렇지만 실제 Vagrant 구성에서 `WIFI` 변수 설정여부에 따라 Neutron external Network를 브릿지 모드로 구성할지 그렇지 않을지가 결정된다. 일반적인 장비에서 브릿지 모드를 활용하기는 어려우므로 우리는 Virtualbox의 NAT Network를 사용할 것이다. 참고로 Virtualbox에서 지원하는 네트워크들의 특성은 아래 표로 나타낼 수 있다. 자세한 설명은 [Virtualbox의 공식 가이드 문서](https://www.virtualbox.org/manual/ch06.html)를 참고하자. 

| Mode | VM -> Host | VM <- Host | VM1 <-> VM2 | VM -> Net/LAN | VM <- Net/LAN |
|--|--|--|--|--|--|
| Host-only | O | O | O | X | X |
| Internal | X | X | O | X | X |
| Bridged | O | O | O | O | O |
| NAT | O | O<br>(Port Forward) | X | O | O<br>(Port Forward) |
| NAT Service | O | O<br>(Port Forward) | O | O | O<br>(Port Forward) |

Vagrant에서 Virtualbox  NATNetwork를 사용할 수 있도록 다음과 같이 NAT Network를 생성해주자. 여기서는 Vagrantfile에서 지정된 대로 NAT Network의 이름을 "OSNetwork"로 생성하였다. 만약 다른 이름을 사용하고 싶다면 Vagrantfile을 수정해야 한다. IP 대역은 임의로 지정해도 무방하지만 나중에 Neutron의 외부 Network로 등록할 때 필요하므로 기억해두도록 하자.

```bash
VBoxManage natnetwork add --netname OSNetwork \
    --network "133.186.0.0/24" \
    --enable --dhcp off
```

마지막으로 operator 노드의 기본 디스크 크기를 다음과 같이 조정하자. 이 operator 노드에서 Kolla를 통해 docker 이미지들을 생성하는데 이 때 디스크 크기가 부족하게 되는 경우가 있었다. 기본 크기가 10GB인데 자신의 환경에서 가용한 크기 내에서 이보다는 큰 값으로 설정해야 한다. 여기서는 적당히 50GB로 설정하였다.

```diff
      if PROVIDER == "libvirt"
        vm.graphics_ip = GRAPHICSIP
      end
      configure_wifi_if_enabled(vm)
    end
    admin.hostmanager.aliases = "operator"
+   admin.disksize.size = "50GB"
  end
```

### Bootstrap 스크립트 수정

Kolla-ansible에서 제공되는 bootstrap.sh에는 몇 가지 수정이 필요한 항목이 있다. 첫 번째는 docker 설치하는 부분이다.  원래 코드에는 `docker-engine`을 설치하는데 그냥 진행하면 제대로 설치가 되지 않는다. 아래 코드를 반영해서 `docker-ce`를 설치하도록 하자.

```diff
function cleanup {
 # Install and configure a quick&dirty docker daemon.
 function install_docker {
     if is_centos; then
-        cat >/etc/yum.repos.d/docker.repo <<-EOF
-[dockerrepo]
-name=Docker Repository
-baseurl=https://mirrors.mediatemple.net/docker/centos/7
-enabled=1
-gpgcheck=1
-gpgkey=https://mirrors.aliyun.com/docker-engine/yum/gpg
-EOF
+        yum -y install yum-utils
+        yum-config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo
+
         # Also upgrade device-mapper here because of:
         # https://github.com/docker/docker/issues/12108
         # Upgrade lvm2 to get device-mapper installed
-        yum -y install docker-engine lvm2 device-mapper
+        yum -y install docker-ce lvm2 device-mapper
 
         # Despite it shipping with /etc/sysconfig/docker, Docker is not configured to
         # load it from it's service file.
-        sed -i -r "s|(ExecStart)=(.+)|\1=/usr/bin/docker daemon --insecure-registry ${REGISTRY} --registry-mirror=http://${REGISTRY}|" /usr/lib/systemd/system/docker.service
+        mkdir -p /etc/docker
+        cat >/etc/docker/daemon.json <<EOF
+{
+    "insecure-registries": ["${REGISTRY}"],
+    "registry-mirrors": ["http://${REGISTRY}"]
+}
+EOF
         sed -i 's|^MountFlags=.*|MountFlags=shared|' /usr/lib/systemd/system/docker.service
 
         usermod -aG docker vagrant
     elif is_ubuntu; then
-        apt-key adv --keyserver hkp://pgp.mit.edu:80 --recv-keys 58118E89F3A912897C070ADBF76221572C52609D
-        echo "deb https://mirrors.aliyun.com/docker-engine/apt/repo ubuntu-xenial main" > /etc/apt/sources.list.d/docker.list
-        apt-get update
-        apt-get -y install docker-engine
-        sed -i -r "s|(ExecStart)=(.+)|\1=/usr/bin/docker daemon --insecure-registry ${REGISTRY} --registry-mirror=http://${REGISTRY}|" /lib/systemd/system/docker.service
+        apt install apt-transport-https ca-certificates curl software-properties-common
+        curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -
+        apt-key fingerprint 0EBFCD88
+        add-apt-repository "deb [arch=amd64] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable"
+        apt update -y
+        apt-get -y install docker.io
+        sed -i -r "s|(ExecStart)=(.+)|\1=/usr/bin/dockerd --insecure-registry ${REGISTRY} --registry-mirror=http://${REGISTRY} --host fd:// --containerd=/run/containerd/containerd.sock|" /lib/systemd/system/doc
+        #apt install docker-ce -y
+        #sed -i -r "s|(ExecStart)=(.+)|\1=/usr/bin/docker daemon --insecure-registry ${REGISTRY} --registry-mirror=http://${REGISTRY}|" /lib/systemd/system/docker.service
     else
         echo "Unsupported Distro: $DISTRO" 1>&2
         exit 1
```

또한 docker 서비스 설치가 끝난 후 정상적으로 사설 repository (operator 노드에서 구동되는) 사설 docker 이미지 repository를 사용할 수 있도록 docker 서비스 재기동이 필요하다. 아래의 코드를 수정해 두자.

```diff
    systemctl daemon-reload
    systemctl enable docker
-   systemctl start docker
+   systemctl restart docker
```

### Vagrant 실행 및 가상 머신 점검

이제 모든 준비가 끝났다. 다음 명령어를 실행하여 Vagrant를 통해 가상 머신들을 구동하자. 명령어를 실행한 후 수 분에서 수십 분이 흐르면 완료된다. 모든 가상 머신들이 다 구동 되었다면 operator 노드로 접근하자.

```bash
$ vagrant up
...
$ vagrant status

Current machine states:

operator                  running (virtualbox)
compute01                 running (virtualbox)
compute02                 running (virtualbox)
storage01                 running (virtualbox)
network01                 running (virtualbox)
control01                 running (virtualbox)
control02                 running (virtualbox)

This environment represents multiple VMs. The VMs are all listed
above with their current state. For more information about a specific
VM, run `vagrant status NAME`.
$ vagrant ssh operator
Welcome to Ubuntu 16.04.7 LTS (GNU/Linux 4.4.0-193-generic x86_64)

 * Documentation:  https://help.ubuntu.com
 * Management:     https://landscape.canonical.com
 * Support:        https://ubuntu.com/advantage

4 packages can be updated.
0 updates are security updates.

Last login: Tue Oct 20 07:07:30 2020 from 10.0.2.2
vagrant@operator:~$
```

본격적으로 kolla-ansible을 실행하기 전에 몇 가지 확인 작업이 더 필요하다. 먼저 ansible이 다른 노드로 접근할 수 있도록 노드 간 ssh 접근을 확인해야 한다. 이는 정상적으로 네트워크가 구성 되었는지 확인이 필요할 뿐만 아니라 노드 별 호스트 키를 ssh의 알려진 노드로 등록해야 때문이다. 앞 서 나온 `[bootstrap.sh](http://bootstrap.sh)` 스크립트에서 ssh 키페어를 생성하여 root 계정 설정에 넣어 두기 때문에 아래 예제와 같이 root 계정으로 실행해야 한다.

```bash
$ sudo su
# ssh control01
...
# ssh control02
...
# ssh compute01
...
# ssh compute02
...
# ssh network01
The authenticity of host 'network01 (172.28.128.23)' can't be established.
ECDSA key fingerprint is SHA256:N1UKRYQBYRSSC4VET4nj+YICK7spmvAnFAeTwqYnRxk.
Are you sure you want to continue connecting (yes/no)? yes
Warning: Permanently added 'network01' (ECDSA) to the list of known hosts.
Welcome to Ubuntu 16.04.7 LTS (GNU/Linux 4.4.0-193-generic x86_64)

 * Documentation:  https://help.ubuntu.com
 * Management:     https://landscape.canonical.com
 * Support:        https://ubuntu.com/advantage

4 packages can be updated.
0 updates are security updates.

Last login: Thu Oct 22 02:01:18 2020 from 172.28.128.19
root@network01:~#
```

또한 각 노드 별로 docker 서비스에 사설 repository를 사용할 수 있도록 등록되어 있는지 확인한다. Docker 서비스의 명령행 인자로 `--insecure-registry`, `--registry-mirror`가 포함되어 있어야 한다.

```bash
# systemctl status docker.service
* docker.service - Docker Application Container Engine
   Loaded: loaded (/lib/systemd/system/docker.service; enabled; vendor preset: enabled)
   Active: active (running) since Thu 2020-10-22 01:44:27 UTC; 1 weeks 3 days ago
     Docs: https://docs.docker.com
 Main PID: 12577 (dockerd)
    Tasks: 19
   Memory: 206.1M
      CPU: 9min 19.946s
   CGroup: /system.slice/docker.service
           `-12577 /usr/bin/dockerd --insecure-registry operator.local:5000 --registry-mirror=http://operator.local:5000 --host fd:// --containerd=/run/containerd/containerd.sock
```

> 이후 다른 명령어는 ubuntu 계정으로 실행해야 한다.

### Ansible 레시피 수정

이 문서에서 진행하는 kolla-ansible는 OpenStack Queens에 대응하는 버전으로서 현 시점에서는 좀 오래된 버전이다. 그러다 보니 기본 제공되는 레시피의 문법 중 일부는 더 이상 지원하지 않는다. 대표적으로 `| changed`, `| success`가 있다. Ansible의 레시피 정의는 operator 노드의 `/usr/local/share/kolla-ansible/ansible` 하위에 있다. 이 경로에 있는 모든 YAML 파일의 `| changed`, `| success` 패턴에 대해 다음과 같이 수정한다.

```diff
-    - horizon_config_json | changed
+    - horizon_config_json is changed
...
-  until: check_keystone_ssh_port | success
+  until: check_keystone_ssh_port is success
...

```

또한 `issuperset` 도 더 이상 지원하지 않는다. 다음과 같이 수정하자.

```diff
-      | issuperset(expected_compute_service_hosts)
+      is superset(expected_compute_service_hosts)
```

마지막으로 Python2 미지원 경고 문구를 제거할 수 있도록 다음과 같이 환경 변수 설정을 추가하자.

```diff
--- kolla-ansible/ansible/roles/common/defaults/main.yml	2020-10-21 05:54:17.792578544 +0000
+++ /usr/local/share/kolla-ansible/ansible/roles/common/defaults/main.yml	2020-10-15 15:41:06.656578544 +0000
@@ -21,6 +21,7 @@
     environment:
       ANSIBLE_NOCOLOR: "1"
       ANSIBLE_LIBRARY: "/usr/share/ansible"
+      PYTHONWARNINGS: "ignore::UserWarning"
```

### Dockerfile 수정

Kolla-ansible이 실행될 때 사용하는 `kolla-toolbox`라는 이미지가 있다. 이 이미지를 빌드할 때 특정 패키지의 버전이 낮아 설치에 실패하는 경우가 있다. 이를 방지하기 위해 `kolla-toolbox`에 대한 Dockerfile을 다음과 같이 수정하자.

```diff
diff -urN kolla/docker/kolla-toolbox/Dockerfile.j2 /usr/local/share/kolla/docker/kolla-toolbox/Dockerfile.j2
--- kolla/docker/kolla-toolbox/Dockerfile.j2    2020-10-21 10:59:37.944578544 +0000
+++ /usr/local/share/kolla/docker/kolla-toolbox/Dockerfile.j2   2020-10-15 16:18:20.832578544 +0000
@@ -81,14 +81,14 @@
         'ansible==2.2.0.0',
         '"cmd2<0.9.0"',
         'MySQL-python',
-        'os-client-config==1.28.0',
+        'os-client-config==1.29.0',
         '"openstacksdk<0.18"',
-        'pbr==2.0.0',
+        'pbr==4.0.0',
         'pymongo',
-        'python-openstackclient==3.12.0',
+        'python-openstackclient==3.14.0',
         'pytz',
         'pyudev',
-        'shade==1.16.0'
+        'shade==1.27.1'
     ] %}
```

이제 Kolla를 실행할 준비가 끝났다. 물론 아직 Kolla-ansible을 통해 OpenStack 설치를 진행하기 까지 필요한 과정은 더 남았긴 하지만 말이다.

## 이미지 빌드하기

Kolla-ansible을 통해 OpenStack을 설치하려면 먼저 필요한 이미지들을 빌드해야 한다. 이 문서에서는 가장 기본이 되는 Keystone, Nova, Neutron, Glance, Heat 정도만 설치한다. 물론 Kolla/Kolla-ansible 설정을 수정하여 다른 프로젝트들도 활성화할 수 있다. 하지만 그 부분은 여러분들의 몫으로 남겨두겠다.

먼저 빌드 옵션을 수정하자. 빌드 옵션은 operator 노드의 `/etc/kolla/kolla-build.conf`에 있다. 해당 파일의 내용 중 일부 항목을 다음과 같이 수정하자. 

```bash
base = ubuntu
profile = default
regex = 
tag = 6.2.5
default = chrony,cron,kolla-toolbox,fluentd,glance,haproxy,heat,horizon,keepalived,keystone,mariadb,memcached,neutron,nova-,openvswitch,rabbitmq
```

설정 파일을 수정한 후 닫고 나와 다음 명령어를 실행한다.

```bash
kolla-build
```

혹은 아래와 같이 빌드할 이미지를 지정할 수도 있다.

```bash
kolla-build keystone
```

한참을 기다리면 이미지가 모두 성공적으로 빌드되었다고 할 것이다. 혹시 중간에 에러가 발생한 이미지는 없는지 살펴보도록 하자.

## 실행하기

자 이제 거의 다왔다. Ansible을 실행하기 전 마지막으로 OpenStack 구성 설정을 해줘야 한다. 아래처럼 `/etc/kolla/globals.yaml` 파일을 수정하자.

```bash
kolla_base_distro: "ubuntu"
openstack_release: "6.2.5"
kolla_internal_vip_address: "172.28.128.254"
network_interface: "enp0s8"
neutron_external_interface: "enp0s9"
neutron_plugin_agent: "openvswitch"
enable_fluentd: "no"
```

위 설정에서 `fluentd`설치를 비활성화하였다. 이는 이후 컨테이너 구동 단계에서 동작하지 않기 때문이었다. 무책임하지만 이 문서에서는 그냥 넘어가기로 하였다.

다음으로 `/usr/local/share/kolla-ansible/ansible/inventory/multinode` 파일을 열어 노드 현황을 업데이트 하자.

```bash
# These initial groups are the only groups required to be modified. The
# additional groups are for more control of the environment.
[control]
# These hostname must be resolvable from your deployment host
control01
control02
#control03

# The above can also be specified as follows:
#control[01:03]     ansible_user=kolla

# The network nodes are where your l3-agent and loadbalancers will run
# This can be the same as a host in the control group
[network]
network01
#network02

# inner-compute is the groups of compute nodes which do not have
# external reachability
[inner-compute]

# external-compute is the groups of compute nodes which can reach
# outside
[external-compute]
compute01
compute02

[compute:children]
inner-compute
external-compute

[monitoring]
#monitoring01
```

그 다음은 OpenStack 설치할 때 사용하는 패스워드 설정이 필요하다. 간단하게 다음 명령을 실행하자.

```bash
kolla-genpwd
```

이제 정말 모든 준비가 다 끝났다. 아래의 명령을 통해 `kolla`를 실행해보자. 억겁의 시간이 흐른 뒤 완료된다.

```bash
sudo kolla-ansible deploy -i /usr/local/share/kolla-ansible/ansible/inventory/multinode
```

## 검증하기

뭔가 스크립트 상으로 설치는 끝났다고 나온다. 정말 제대로 설치된 것인가? 정상 설치 여부를 확인하기 위해 Neutron의 네트워크를 구성하고 Glance 이미지를 준비한 후 Nova의 키페어/Flavor를 생성한 뒤, 이를 이용해 Nova 인스턴스를 생성하고 접속 테스트를 할 것이다.

다행히도 Kolla-ansible에서는 이러한 상황에 사용할 수 있는 스크립트를 미리 준비되었다. 이 스크립트를 실행하기 위해서는 앞서 설정한 외부 네트워크 대역을 인식할 수 있도록 약간의 수정이 필요하다. Operator 노드의 `/usr/local/share/kolla-ansible/init-runonce`파일을 열어 다음과 같이 수정하자.

```bash
# This EXT_NET_CIDR is your public network,that you want to connect to the internet via.
EXT_NET_CIDR='133.186.0.0/24'
EXT_NET_RANGE='start=133.186.0.150,end=133.186.0.199'
EXT_NET_GATEWAY='133.186.0.1'
```

검증 스크립트를 실행하기 전 다음과 같이 OpenStack 환경 변수를 설정한다.

```bash
source /etc/kolla/admin-openrc.sh
```

이제 검증 스크립트를 실행하면 자동으로 네트워크를 구성하고 Nova 인스턴스가 생성될 것이다.

```bash
$ /usr/local/share/kolla-ansible/init-runonce
...
$ nova list
+--------------------------------------+-------+--------+------------+-------------+--------------------+
| ID                                   | Name  | Status | Task State | Power State | Networks           |
+--------------------------------------+-------+--------+------------+-------------+--------------------+
| 851c59cc-ea40-40b1-92e1-7acd0cb548c4 | demo1 | ACTIVE | -          | Running     | demo-net=10.0.0.18 |
+--------------------------------------+-------+--------+------------+-------------+--------------------+
```

보다시피 아직 인스턴스에 접근하기 위한 Floating IP가 없는 상태이다. 다음과 같이 Floating IP를 생성하고 인스턴스에 붙여주자.

```bash
PUBLIC_NET_ID=`neutron net-list | grep public1 | awk '{print $2}'`
NOVA_INSTANCE_ID=`nova list | grep demo1 | awk '{print $2}'`
FLOATING_IP_ADDRESS=`neutron floatingip-create $PUBLIC_NET_ID | grep floating_ip_address | awk '{print $4}'`
openstack server add floating ip $NOVA_INSTANCE_ID $FLOATING_IP_ADDRESS
```

인스턴스에 접근하기 위해서는 호스트 장비에서 포트 포워딩을 통해 접근해야 한다. 새로운 터미널 창을 띄우고 다음과 같이 Virtualbox 설정을 하자. 아래의 명령은 호스트 장비의 13310 포트에서 NAT Network의 133.186.0.10의 22번 포트로 접근하도록 설정한 것이다. 이 때 목적지 IP는 앞서 받은 Floating IP 주소로 해야 한다.

```bash
vboxmanage natnetwork modify --netname OSNetwork --port-forward-4 "ssh:tcp:[]:13310:[133.186.0.10]:22"
```

이제 인스턴스에 접근해보자. 정상적으로 Cirros 프롬프트가 뜬다면 성공한 것이다. 축하한다.

```bash
ssh -p 13310 localhost
```
