=======================================
Burrito Online 친절한 설치가이드
=======================================


.. This content will be ignored during compilation
   .. contents::
      :local:
      :backlinks: none
      :depth: 2



지원 OS
---------------

* Rocky Linux 8.x




네트워크
-----------

Burrito에서 기본적으로 사용하는 네트워크는 총 5개입니다.

* service network: 웹 포탈 서비스 / 서버 접속 용도 네트워크 (예: 192.168.20.0/24)
* management network: K8S, Openstack 관리 네트워크 (예: 192.168.21.0/24)
* provider network: 가상 PC에 할당할 네트워크 (예: 192.168.22.0/24)
* overlay network: OpenStack 오버레이 네트워크 (예: 192.168.23.0/24)
* storage network: Ceph Public(Mon)/Cluster(OSD) 네트워크 (예: 192.168.24.0/24)

네트워크 구조 예시
++++++++++++++++++++

아래는 네트워크 설계 가이드 예시입니다. 

* control/compute 머신은 5개의 네트워크를 모두 가지고 있습니다.
* provider network는 네트워크에는 IP 주소가 할당되지 않습니다.
    * 단, btx 환경에서 VM 생성 시 네트워크가 필요합니다. `BTX --test 참조 <#test-section>`_
* storage 머신은 2개의 네트워크를 가지고 있습니다. (management and storage)

기본 네트워크 예시
^^^^^^^^^^^^^^^^^^^^^^^

========  ============ ============ ============ ============ ============
hostname  service      management   provider     overlay      storage
--------  ------------ ------------ ------------ ------------ ------------
 .        eth0         eth1         eth2         eth3         eth4
 .        192.168.20.x 192.168.21.x 192.168.22.x 192.168.23.x 192.168.24.x 
========  ============ ============ ============ ============ ============
control1  .101          .101          (no ip)     .101           .101
control2  .102          .102          (no ip)     .102           .102
control3  .103          .103          (no ip)     .103           .103
compute1  .104          .104          (no ip)     .104           .104
compute2  .105          .105          (no ip)     .105           .105
storage1                .106                                     .106
storage2                .107                                     .107
storage3                .108                                     .108
========  ============ ============ ============ ============ ============

기타 네트워크 예시
^^^^^^^^^^^^^^^^^^^^^^^

=================  ================  ================  ================  ================  ================
분류               service           management        provider          overlay           storage
-----------------  ----------------  ----------------  ----------------  ----------------  ----------------
CIDR               192.168.20.0/24   192.168.21.0/24   192.168.22.0/24   192.168.23.0/24   192.168.24.0/24
-----------------  ----------------  ----------------  ----------------  ----------------  ----------------
metalLB            192.168.20.0/24  
-----------------  ----------------  ----------------  ----------------  ----------------  ----------------
KeepAlived VIP     192.168.20.100    192.168.21.100
=================  ================  ================  ================  ================  ================


정의
---------


* Burrito ISO 파일을 가져와야 합니다.
* control 그룹에서 첫번째 노드(control1)가 ansible 배포 노드입니다.
* 모든 노드에 ansible 사용자는 sudo 권한이 있다. ansible 사용자는 clex 입니다.
* 모든 노드는 배포 노드의 /etc/hosts에 정의되어야 한다.

.. attention:: 

   ::
   
      control 노드 1번 에서 /etc/hosts를 정의합니다.

      management 네트워크 대역 IP로 기재되어야 합니다.

::

   127.0.0.1 localhost
   192.168.21.101 control1
   192.168.21.102 control2 
   192.168.21.103 control3 
   192.168.21.104 compute1 
   192.168.21.105 compute2 
   192.168.21.106 storage1 
   192.168.21.107 storage2 
   192.168.21.108 storage3 



준비
--------

iso 파일을 mount 한다.

::

   $ sudo mount -o loop,ro <path/to/burrito_iso_file> /mnt

tar 압축 파일을 mnt 디렉토리에서 확인한다.

::

   $ ls /mnt/burrito-*.tar.gz
   /mnt/burrito-<version>.tar.gz

burrito 압축 파일을 홈디렉토리에서 압축 해제한다.

::

   $ tar xzf /mnt/burrito-<version>.tar.gz

이제 burrito 디렉토리로 들어간다.

::

   $ cd burrito-<version>

이제 prepare shell script를 실행한다.

그리고 management 네트워크 이름을 입력한다. (예: eth1)

::
   

   $ ./prepare.sh 
   





인벤토리 호스트
++++++++++++++++++++++++++++

Inventory hosts는 Ansible에서 사용되는 호스트(서버, 가상 머신, 네트워크 장비 등)의 목록을 정의하는 파일 또는 그룹이다.

이 파일은 Ansible이 작업을 수행할 대상 호스트를 식별하고 선택하는 데 사용한다.

burrito 4개의 호스트 그룹
^^^^^^^^^^^^^^^^^^^^^^^^^^^^

* 컨트롤 노드(Control Node): 쿠버네티스(Kubernetes)와 오픈스택(OpenStack)의 제어 구성 요소를 실행
* 네트워크 노드(Network Node): 쿠버네티스 워커(Worker) 및 오픈스택 네트워크 서비스를 실행
   * 네트워크 노드는 선택 사항이다. 일반적으로 Control 노드가 Control 노드와 네트워크 노드의 역할을 겸한다.
* 컴퓨트 노드(Compute Node): 쿠버네티스 워커 및 오픈스택 하이퍼바이저(Hypervisor)와 네트워크 에이전트(Network Agent)를 실행하여 인스턴스를 운영
* 스토리지 노드(Storage Node): Ceph storage 서비스인 모니터(Monitor), 매니저(Manager), OSD, Rados 게이트웨이를 실행



.. attention::

   ::
   
      hosts 파일 편집 시 주의사항     

      1. 네트워크 노드가 따로 없으므로 control 노드를 네트워크 노드 그룹에 넣으면 됩니다.

      2. 반드시 etc/hosts 정의된 내용을 참조하여 작성합니다.

      3. Do not edit below 아래의 내용은 건드리지 않습니다. 


hosts 인벤토리 파일 편집합니다.

::

   $ vi hosts
   control1 ip=192.168.21.101 ansible_connection=local ansible_python_interpreter=/usr/bin/python3
   control2 ip=192.168.21.102
   control3 ip=192.168.21.103
   compute1 ip=192.168.21.104
   compute2 ip=192.168.21.105
   storage1 ip=192.168.21.106 monitor_address=192.168.24.106 radosgw_address=192.168.24.106
   storage2 ip=192.168.21.107 monitor_address=192.168.24.107 radosgw_address=192.168.24.107
   storage3 ip=192.168.21.108 monitor_address=192.168.24.108 radosgw_address=192.168.24.108

   # ceph nodes
   [mons]
   storage[1:3]
   
   [mgrs]
   storage[1:3]
   
   [osds]
   storage[1:3]
   
   [rgws]
   storage[1:3]
   
   [clients]
   control[1:3]
   compute[1:2]
   
   # kubernetes nodes
   [kube_control_plane]
   control[1:3]
   
   [kube_node]
   control[1:3]
   compute[1:2]
   
   # openstack nodes
   [controller-node]
   control[1:3]
   
   [network-node]
   control[1:3]
   
   [compute-node]
   compute[1:2]
   
   ###################################################
   ## Do not touch below if you are not an expert!!! #
   ###################################################

인벤토리 변수
++++++++++++++++++++++++++++

.. attention::

   ::

      hosts 파일 편집 시 주의사항


      1. 바꿔야 하는 변수("""내용"""표시)만 바꿔주고 다른 변수나 Do not edit below는 건드리지 않습니다. 

      2. 변수에 대한 설명을 반드시 읽고 수정합니다.

      3. storage가 1개인 경우 1개만 작성해야 하고 2개인 경우 default를 첫번째 순서로 작성합니다.

      4. """내용""" 표시는 변수에 대한 설명이 되어 있는 부분입니다.

vars yml파일을 편집합니다.::

   $ vi vars.yml
   ---
   ### common
   # deploy_ssh_key: (boolean) create ssh keypair and copy it to other nodes.
   # default: false
   deploy_ssh_key: false

   """
   deploy_ssh_key (default: false)
   true인 경우 배포자 노드에 ssh 키 쌍을 생성하고 공개 키를 다른 노드에 복사합니다. 
   Ansible은 공개 키를 배포한 후 공개 키를 사용하여 다른 노드에 ssh합니다.
   false인 경우 ssh 키 쌍을 생성하지 않습니다. 
   Ansible은 볼트로 암호화된 사용자의 비밀번호를 사용하여 다른 노드로 ssh합니다.
   """
   
   ### define network interface names
   # set overlay_iface_name to null if you do not want to set up overlay network.
   # then, only provider network will be set up.
   svc_iface_name: eth0
   mgmt_iface_name: eth1
   provider_iface_name: eth2
   overlay_iface_name: eth3
   storage_iface_name: eth4   

   """
   iface_name
   각 네트워크 인터페이스 이름을 설정합니다.
   provider 네트워크만 설정한다면 overlay_iface_name을 null로 설정합니다. 
   overlay 네트워크가 없다면 openstack neutron 네트워크 서비스는 비활성화 됩니다.   
   """

   ### ntp
   # Specify time servers for control nodes.
   # You can use the default ntp.org servers or time servers in your network.
   # If servers are offline and there is no time server in your network,
   #   set ntp_servers to empty list.
   #   Then, the control nodes will be the ntp servers for other nodes.
   # ntp_servers: []
   ntp_servers:
     - 0.pool.ntp.org
     - 1.pool.ntp.org
     - 2.pool.ntp.org
   
   """
   ntp_servers (default: {0,1,2}.pool.ntp.org)
   control 노드에 대한 시간 서버를 지정해야 합니다.
   네트워크에서 기본 ntp.org 서버 또는 시간 서버를 사용할 수 있습니다.
   만약 서버가 오프라인이고 네트워크에 시간 서버가 없으면 ntp_servers를 빈 목록(ntp_servers: [])으로 설정합니다. 
   그렇게되면 control 노드는 다른 노드의 ntp 서버가 됩니다.
   """
   
   ### keepalived VIP on management network (mandatory)
   keepalived_vip: ""
   # keepalived VIP on service network (optional)
   # Set this if you do not have a direct access to management network
   # so you need to access horizon dashboard through service network.
   keepalived_vip_svc: ""

   """
   keepalived_vip (필수)
   LoadBalancing 및 내부 서비스에 대한 고가용성을 위해 management 네트워크의 VIP 주소를 할당합니다.
   필수이므로 반드시 작성해야 합니다.
   keepalived_vip_svc (선택)
   Horizon 대시보드 서비스를 위해 service 네트워크에 VIP 주소를 할당합니다. 
   management 네트워크에 직접 액세스할 수 없는 경우 설정합니다.
   할당되지 않은 경우 management 네트워크에서 keepalived_vip를 통해 Horizon 대시보드에 연결해야 합니다.
   """

   
   ### metallb
   # To use metallb LoadBalancer, set this to true
   metallb_enabled: false
   # set up MetalLB LoadBalancer IP range or cidr notation
   # IP range: 192.168.20.95-192.168.20.98 (4 IPs can be assigned.)
   # CIDR: 192.168.20.128/26 (192.168.20.128 - 191 can be assigned.)
   # Only one IP: 192.168.20.95/32
   metallb_ip_range:
     - "192.168.20.95-192.168.20.98"

   """
   metallb_enabled (default: false)
   metallb LoadBalancer를 사용하려면 true로 설정해야 합니다.
   (참조 ` metallb에 대해 알고 싶다면 <https://metallb.universe.tf/>`_)
   metallb_ip_range
   metallb LoadBalancer IP 범위 또는 cidr 표기법을 설정합니다.
   * IP 범위: 192.168.20.95-192.168.20.98(4개의 IP 할당 가능)
   * CIDR: 192.168.20.128/26(192.168.20.128 - 191 지정 가능)
   * 하나의 IP: 192.168.20.95/32(192.168.20.95 할당 가능)
   IP 범위 또는 cidr 표기법 정했다면 해당 변수만 수정합니다.
   metallb_ip_range: 
   - "이 곳에만 작성합니다."
   """
    

   ### storage
   # storage backends: ceph and(or) netapp
   # If there are multiple backends, the first one is the default backend.
   storage_backends:
     - netapp
     - ceph
   
   # ceph: set ceph configuration in group_vars/all/ceph_vars.yml
   # netapp: set netapp configuration in group_vars/all/netapp_vars.yml

   """
   storage_backends
   Burrito는 ceph 와 netapp 두 가지 storage 백엔드를 지원합니다.
   백엔드가 여러 개인 경우 첫 번째 백엔드가 기본 백엔드입니다. 
   이는 기본 storageclass, gladiator store 및 기본 cinder 볼륨 유형이 첫 번째 백엔드임을 의미합니다.
   storageclass 이름을 지정하지 않으면 영구 볼륨이 기본 백엔드에 생성됩니다.
   볼륨 유형을 지정하지 않으면 기본 볼륨 유형에 볼륨이 생성됩니다.
   추가적으로 storage 변수 설정은 burrito-<version>/group_vars/all 경로에서 수정합니다.
   """

   ###################################################
   ## Do not edit below if you are not an expert!!!  #
   ###################################################



storage 변수 설정
^^^^^^^^^^^^^^^^^^^^^^

storage 변수 설정에서는 group_vars/all/ceph_vars.yml 또는 group_vars/all/netapp_vars.yml 편집합니다.

*ceph*
^^^^^^^^^^

ceph가 storage_backends에 있는 경우 storage 노드에서 lsblk 명령을 실행하여 장치 이름을 가져옵니다.

이 경우 /dev/sda는 OS 디스크이고 /dev/sd{b,c,d}는 ceph OSD 디스크용입니다.

::


   storage1$ lsblk -p
   NAME        MAJ:MIN RM SIZE RO TYPE MOUNTPOINT
   /dev/sda      8:0    0  50G  0 disk 
   └─/dev/sda1   8:1    0  50G  0 part /
   /dev/sdb      8:16   0  50G  0 disk 
   /dev/sdc      8:32   0  50G  0 disk 
   /dev/sdd      8:48   0  50G  0 disk 



group_vars/all/ceph_vars.yml을 편집하고 /dev/sd{b,c,d}를 추가합니다.

::

   $ vi group_vars/all/ceph_vars.yml
   ---
   # ceph config
   lvm_volumes:
     - data: /dev/sdb
     - data: /dev/sdc
     - data: /dev/sdd
   ...

*netapp*
^^^^^^^^^^^^^

netapp이 storage_backends에 있는 경우 group_vars/all/netapp_vars.yml을 편집합니다.

netapp 각각의 변수가 무엇인지 모르는 경우 netapp 엔지니어에게 도움을 구하세요.

::

   $ vi group_vars/all/netapp_vars.yml
   ---
   netapp:
     - name: netapp1
       managementLIF: "192.168.100.230"
       dataLIF: "192.168.140.19"
       svm: "svm01"
       username: "admin"
       password: "<netapp_admin_password>"
       nfsMountOptions: "nfsvers=4,lookupcache=pos"
       shares:
         - /dev03
   ...




볼트 파일 설정
^^^^^^^^^^^^^^^^

다른 노드에 대한 ssh 연결을 위한 <user> 암호를 입력합니다.

openstack Horizon 대시보드에 연결할 때 사용할 openstack 관리자 암호를 입력합니다.

암호를 암호화할 볼트 파일을 만듭니다.::

   $ ./run.sh vault
   <user> password:
   openstack admin password:
   Encryption successful




모든 노드 네트워크 연결 확인
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^


다른 노드에 대한 연결을 확인합니다.

::

   $ ./run.sh ping

.. attention::

   ::

      모든 노드에 SUCCESS가 표시되어야 합니다.



설치
--------

playbook이 실행될 때마다 PLAY RECAP 에 실패(fail) 작업이 없어야 합니다.

예시::

   PLAY RECAP *****************************************************************
   control1                   : ok=20   changed=8    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0   
   control2                   : ok=19   changed=8    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0   
   control3                   : ok=19   changed=8    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0   

.. attention::

   ::


      각 단계마다 인증 절차가 있으므로 다음 단계로 진행하기 전에 반드시 확인해야 합니다.

      확인에 실패하면 절대 다음 단계로 진행하지 마세요.



Step.1 Preflight
+++++++++++++++++

Preflight 설치 단계는 다음 작업을 합니다.

* local yum 저장소를 설정합니다.
* NTP 시간 서버 및 클라이언트를 구성합니다.
* 공개 ssh 키를 다른 노드에 배포합니다(deploy_ssh_key가 true인 경우).

설치
^^^^^^^

preflight playbook 실행합니다.

::

   $ ./run.sh preflight




ntp 서버와 클라이언트가 구성되어 있는지 확인합니다.

ntp_servers를 빈 목록(ntp_servers: [])으로 설정하면 각 control 노드에는 다른 control 노드의 시간 서버가 있어야 합니다.

::

   control1$ chronyc sources
   MS Name/IP address      Stratum Poll Reach LastRx Last sample               
   ========================================================================
   ^? control2             9   6   377   491   +397ms[ +397ms] +/-  382us
   ^? control3             9   6   377   490   -409ms[ -409ms] +/-  215us


compute/storage 노드에는 control 노드가 시간 서버로 있어야 합니다.

::

   $ chronyc sources
   MS Name/IP address      Stratum Poll Reach LastRx Last sample               
   ========================================================================
   ^* control1             8   6   377    46    -15us[  -44us] +/-  212us
   ^- control2             9   6   377    47    -57us[  -86us] +/-  513us
   ^- control3             9   6   377    47    -97us[ -126us] +/-  674us



Step.2 HA 
++++++++++

HA 설치 단계는 다음 작업을 합니다.

* KeepAlived 서비스를 설정합니다.
* HAProxy 서비스를 설정합니다.

KeepAlived 및 HAProxy 서비스는 Burrito 플랫폼의 필수 서비스입니다.

OpenStack 통신, local container registry, local yum repository, ceph Rados 게이트웨이 서비스는 포함됩니다.

여기서 local container registry란 컨테이너 이미지를 저장하고 관리하는 서비스입니다. local 환경에서 컨테이너 이미지를 효율적으로 배포하고 관리합니다.

local yum repository란 Yum 패키지를 사용하여 패키지를 설치하고 업데이트하는 데 사용되는 저장소입니다. 마찬가지로 local 환경에서 필요한 패키지를 사전에 다운로드하여 설치할 수 있도록 합니다.

ceph Rados 게이트웨이 서비스는 RESTful API를 통해 데이터에 접근하며, S3 및 Swift 호환 프로토콜을 지원하여 다양한 애플리케이션과의 통합하는데 용이합니다.


설치
^^^^^^^

HA stack playbook 실행합니다.

::

   $ ./run.sh ha

확인
^^^^^^

keepalived 및 haproxy가 control 노드에서 실행 중인지 확인합니다.

::

   $ sudo systemctl status keepalived haproxy
   keepalived.service - LVS and VRRP High Availability Monitor
   ...
      Active: active (running) since Wed 2023-05-31 17:29:05 KST; 6min ago
   ...
   haproxy.service - HAProxy Load Balancer
   ...
      Active: active (running) since Wed 2023-05-31 17:28:52 KST; 8min ago


첫 번째 control 노드의 management 인터페이스에 keepalived_vip이 생성되었는지 확인합니다.

::

   $ ip -br -4 address show dev eth1
   eth1             UP             192.168.21.101/24 192.168.21.100/32 

설정한 경우 첫 번째 control 노드의 service 인터페이스에 keepalived_vip_svc가 생성되었는지 확인합니다.

::

   $ ip -br -4 address show dev eth0
   eth0             UP             192.168.20.101/24 192.168.20.100/32 


Step.3 Ceph
+++++++++++

.. attention::

   ::

      ceph가 storage_backends에 없으면 이 단계를 건너뜁니다.

      만약 storage_backends 첫번째 순서가 netapp이라도 ceph playbook부터 실행해야 합니다.

Ceph 설치 단계는 다음 작업을 합니다.

* storage 노드에 ceph 서버 및 클라이언트 패키지를 설치합니다.
* 다른 노드에 ceph 클라이언트 패키지를 설치합니다.
* storage 노드에서 ceph 모니터, 관리자, osd, rados 게이트웨이 서비스를 설정합니다.

설치
^^^^^^^

ceph가 storage_backends에 있는 경우 ceph playbook을 실행합니다.

::

   $ ./run.sh ceph

확인
^^^^^^

ceph playbook을 실행한 후 ceph 상태를 확인합니다. HEALTH_OK가 표시되어야 합니다.

::

   $ sudo ceph health
   HEALTH_OK



자세한 상태를 확인하려면 sudo ceph -s 명령을 실행합니다. 다음과 같은 출력이 표시됩니다.

mon, mgr, osd 및 rgw의 4가지 서비스가 있습니다.

::

   $ sudo ceph -s
     cluster:
       id:     cd7bdd5a-1814-4e6a-9e07-c2bdc3f53fea
       health: HEALTH_OK
    
     services:
       mon: 3 daemons, quorum storage1,storage2,storage3 (age 17h)
       mgr: storage2(active, since 17h), standbys: storage1, storage3
       osd: 9 osds: 9 up (since 17h), 9 in (since 17h)
       rgw: 3 daemons active (3 hosts, 1 zones)
    
     data:
       pools:   10 pools, 513 pgs
       objects: 2.54k objects, 7.3 GiB
       usage:   19 GiB used, 431 GiB / 450 GiB avail
       pgs:     513 active+clean



가끔 `HEALTH_WARN <something> have recently crashed` 은 문제가 없을 가능성이 높습니다.

하지만 확인을 위해 `HEALTH_WARN <something> have recently crashed` 상태라면 아래 명령어를 실행합니다.

crash 목록을 조회합니다.

::

   $ sudo ceph crash ls

모든 crash를 archive 합니다.

그런 다음 ceph 상태를 다시 확인합니다. 이제 HEALTH_OK가 표시되어야 합니다.

::

   $ sudo ceph crash archive-all



Step.4 Kubernetes
++++++++++++++++++++

Kubernetes 설치 단계는 다음 작업을 합니다.

* kubernetes 노드에 kubernetes binaries를 설치합니다.
* kubernetes control plane을 설정합니다.
* Kubernetes worker 노드를 설정합니다.
* kube-system namespace에서 local registry를 설정합니다.

설치
^^^^^^^

k8s playbook을 실행합니다.

::

   $ ./run.sh k8s

확인
^^^^^^^

모든 노드가 ready 상태인지 확인합니다.

::

   $ sudo kubectl get nodes
   NAME       STATUS   ROLES           AGE   VERSION
   compute1   Ready    <none>          15m   v1.24.14
   compute2   Ready    <none>          15m   v1.24.14
   control1   Ready    control-plane   17m   v1.24.14
   control2   Ready    control-plane   16m   v1.24.14
   control3   Ready    control-plane   16m   v1.24.14


Step.5 Netapp
++++++++++++++++

.. attention::

   ::

      netapp이 storage_backends에 없다면 이 단계를 건너뜁니다.

Netapp 설치 단계는 다음 작업을 합니다.

* trident namespace에 trident 구성 요소를 설치합니다.
* netapp 백엔드를 설정합니다.
* netapp Storage 클래스를 생성합니다.

설치
^^^^^^^

netapp playbook 실행합니다.

::

   $ ./run.sh netapp


확인
^^^^^^

모든 pod가 실행중이고 trident namespace에서 running 상태인지 확인합니다.

::

   $ sudo kubectl get pods -n trident
   NAME                           READY   STATUS    RESTARTS   AGE
   trident-csi-6b96bb4f87-tw22r   6/6     Running   0          43s
   trident-csi-84g2x              2/2     Running   0          42s
   trident-csi-f6m8w              2/2     Running   0          42s
   trident-csi-klj7h              2/2     Running   0          42s
   trident-csi-kv9mw              2/2     Running   0          42s
   trident-csi-r8gqv              2/2     Running   0          43s

Step.6 Patch
+++++++++++++++

패치 설치 단계는 다음 작업을 합니다.

* ceph가 storage_backends에 있으면 ceph-csi 드라이버를 설치합니다.
* containerd 구성을 패치합니다.
* kube-apiserver를 패치합니다.

설치
^^^^^^^

patch playbook 실행합니다.

::

   $ ./run.sh patch

확인
^^^^^^

패치 후 kube-apiserver를 다시 시작하는데 약간의 시간이 걸립니다.

kube-system namespace에서 모든 pod가 실행중이고 running 상태인지 확인합니다.

.. attention::

   ::

      registry pod가 running 상태가 될때까지 기다려야 합니다.

::

   $ sudo kubectl get pods -n kube-system
   NAME                                       READY STATUS    RESTARTS      AGE
   calico-kube-controllers-67c66cdbfb-rz8lz   1/1   Running   0             60m
   calico-node-28k2c                          1/1   Running   0             60m
   calico-node-7cj6z                          1/1   Running   0             60m
   calico-node-99s5j                          1/1   Running   0             60m
   calico-node-tnmht                          1/1   Running   0             60m
   calico-node-zmpxs                          1/1   Running   0             60m
   coredns-748d85fb6d-c8cj2                   1/1   Running   1 (28s ago)   59m
   coredns-748d85fb6d-gfv98                   1/1   Running   1 (27s ago)   59m
   dns-autoscaler-795478c785-hrjqr            1/1   Running   1 (32s ago)   59m
   kube-apiserver-control1                    1/1   Running   0             33s
   kube-apiserver-control2                    1/1   Running   0             34s
   kube-apiserver-control3                    1/1   Running   0             35s
   kube-controller-manager-control1           1/1   Running   1             62m
   kube-controller-manager-control2           1/1   Running   1             62m
   kube-controller-manager-control3           1/1   Running   1             62m
   kube-proxy-jjq5l                           1/1   Running   0             61m
   kube-proxy-k4kxq                           1/1   Running   0             61m
   kube-proxy-lqtgc                           1/1   Running   0             61m
   kube-proxy-qhdzh                           1/1   Running   0             61m
   kube-proxy-vxrg8                           1/1   Running   0             61m
   kube-scheduler-control1                    1/1   Running   2             62m
   kube-scheduler-control2                    1/1   Running   1             62m
   kube-scheduler-control3                    1/1   Running   1             62m
   nginx-proxy-compute1                       1/1   Running   0             60m
   nginx-proxy-compute2                       1/1   Running   0             60m
   nodelocaldns-5dbbw                         1/1   Running   0             59m
   nodelocaldns-cq2sd                         1/1   Running   0             59m
   nodelocaldns-dzcjr                         1/1   Running   0             59m
   nodelocaldns-plhwm                         1/1   Running   0             59m
   nodelocaldns-vlb8w                         1/1   Running   0             59m
   registry-5v9th                             1/1   Running   0             58m



Step.7 Burrito
+++++++++++++++

Burrito 설치 단계는 다음 작업을 합니다.

* rados 게이트웨이 사용자(기본값: cloudpc) 및 클라이언트 구성(s3cfg)을 생성합니다.
* nova vnc TLS 인증서를 배포합니다.
* openstack 구성 요소를 배포합니다.
* nova ssh 키를 생성하고 모든 compute 노드에 복사합니다.

설치
^^^^^^^

burrito playbook 실행합니다.

::
   $ sudo helm plugin install https://github.com/databus23/helm-diff
   $ ./run.sh burrito

확인
^^^^^^

모든 pod가 실행중이고 openstack namespace에서 running 상태인지 확인합니다.

::

   $ sudo kubectl get pods -n openstack
   NAME                                   READY   STATUS      RESTARTS   AGE
   barbican-api-664986fd5-jkp9x           1/1     Running     0          4m23s
   ...
   rabbitmq-rabbitmq-0                    1/1     Running     0          27m
   rabbitmq-rabbitmq-1                    1/1     Running     0          27m
   rabbitmq-rabbitmq-2                    1/1     Running     0          27m


축하합니다! 당신은 Burrito 플랫폼 설치를 완료했습니다.

이제 Horizon 대시보드를 확인하고 BTX로 가상 머신을 생성할 수 있다.



Horizon
----------

Horizon 대시보드는 control 노드에서 tcp 31000을 수신합니다.

브라우저에서 Horizon 대시보드에 연결하는 방법은 다음과 같습니다.

#. 브라우저를 엽니다.

#. keepalived_vip_svc가 설정되어 있으면 https:/// <keepalived_vip_svc>:31000/ 으로 이동합니다.

#. keepalived_vip_svc가 설정되지 않은 경우 https:/// <keepalived_vip>:31000/ 으로 이동합니다.

#. 자체 서명된 TLS 인증서를 확인하고 로그인합니다. 
   관리자 비밀번호는 vault.sh 스크립트를 실행할 때 설정한 비밀번호입니다.(openstack 관리자 비밀번호)

다음으로 btx(burrito toolbox)를 사용하여 기본 openstack 작동 테스트를 수행합니다.


BTX
-----

BTX는 burrito 플랫폼을 위한 툴박스입니다. 이미 running 상태여야 합니다.

::

   $ sudo kubectl -n openstack get pods -l application=btx
   NAME    READY   STATUS    RESTARTS   AGE
   btx-0   1/1     Running   0          36m

btx 쉘(bts)로 이동합니다.

::

   $ . ~/.btx.env
   $ bts

openstack 볼륨 서비스 상태를 확인합니다.

::

   root@btx-0:/# openstack volume service list
   +------------------+------------------------------+------+---------+-------+----------------------------+
   | Binary           | Host                         | Zone | Status  | State | Updated At                 |
   +------------------+------------------------------+------+---------+-------+----------------------------+
   | cinder-scheduler | cinder-volume-worker         | nova | enabled | up    | 2023-05-31T12:05:02.000000 |
   | cinder-volume    | cinder-volume-worker@rbd1    | nova | enabled | up    | 2023-05-31T12:05:02.000000 |
   | cinder-volume    | cinder-volume-worker@netapp1 | nova | enabled | up    | 2023-05-31T12:05:07.000000 |
   +------------------+------------------------------+------+---------+-------+----------------------------+

* 모든 서비스가 활성화되어 있어야 합니다.
* ceph 및 netapp storage 백엔드를 모두 설정하면 두 볼륨 서비스가 모두 활성화되고 출력에 표시됩니다.
* cinder -volume-worker@rbd1은 ceph 백엔드용 서비스이고 
  cinder-volume-worker@netapp1 은 netapp 백엔드용 서비스입니다.

openstack 네트워크 에이전트 상태를 확인합니다.

::

   root@btx-0:/# openstack network agent list
   +--------------------------------------+--------------------+----------+-------------------+-------+-------+---------------------------+
   | ID                                   | Agent Type         | Host     | Availability Zone | Alive | State | Binary                    |
   +--------------------------------------+--------------------+----------+-------------------+-------+-------+---------------------------+
   | 0b4ddf14-d593-44bb-a0aa-2776dfc20dc9 | Metadata agent     | control1 | None              | :-)   | UP    | neutron-metadata-agent    |
   | 189c6f4a-4fad-4962-8439-0daf400fcae0 | DHCP agent         | control3 | nova              | :-)   | UP    | neutron-dhcp-agent        |
   | 22b0d873-4192-41ad-831b-0d468fa2e411 | Metadata agent     | control3 | None              | :-)   | UP    | neutron-metadata-agent    |
   | 4e51b0a0-e38a-402e-bbbd-5b759130220f | Linux bridge agent | compute1 | None              | :-)   | UP    | neutron-linuxbridge-agent |
   | 56e43554-47bc-45c8-8c46-fb2aa0557cc0 | DHCP agent         | control1 | nova              | :-)   | UP    | neutron-dhcp-agent        |
   | 7f51c2b7-b9e3-4218-9c7b-94076d2b162a | Linux bridge agent | compute2 | None              | :-)   | UP    | neutron-linuxbridge-agent |
   | 95d09bfd-0d71-40d4-a5c2-d46eb640e967 | DHCP agent         | control2 | nova              | :-)   | UP    | neutron-dhcp-agent        |
   | b76707f2-f13c-4f68-b769-fab8043621c7 | Linux bridge agent | control3 | None              | :-)   | UP    | neutron-linuxbridge-agent |
   | c3a6a32c-cbb5-406c-9b2f-de3734234c46 | Linux bridge agent | control1 | None              | :-)   | UP    | neutron-linuxbridge-agent |
   | c7187dc2-eea3-4fb6-a3f6-1919b82ced5b | Linux bridge agent | control2 | None              | :-)   | UP    | neutron-linuxbridge-agent |
   | f0a396d3-8200-41c3-9057-5d609204be3f | Metadata agent     | control2 | None              | :-)   | UP    | neutron-metadata-agent    |
   +--------------------------------------+--------------------+----------+-------------------+-------+-------+---------------------------+

* 모든 에이전트는 :-) 및 UP이어야 합니다.
* overlay_iface_name을 null로 설정하면 Agent Type 열에 'L3 agent'가 없습니다.
* is_ovs를 false로 설정하면 Agent Type 열에 'Linux bridge agent'가 있어야 합니다.
* is_ovs를 true로 설정하면 Agent Type 열에 'Open vSwitch agent'가 있어야 합니다.


openstack compute 서비스 상태를 확인합니다.

::

   root@btx-0:/# openstack compute service list
   +--------------------------------------+----------------+---------------------------------+----------+---------+-------+----------------------------+
   | ID                                   | Binary         | Host                            | Zone     | Status  | State | Updated At                 |
   +--------------------------------------+----------------+---------------------------------+----------+---------+-------+----------------------------+
   | b31c814b-d210-4e52-9d6e-59090f8a641a | nova-scheduler | nova-scheduler-5bcc764f79-wkfgl | internal | enabled | up    | 2023-05-31T12:16:20.000000 |
   | 872555ad-dd52-46ce-be01-1ec7f8af9cd9 | nova-conductor | nova-conductor-56dfd9749-fn9xb  | internal | enabled | up    | 2023-05-31T12:16:21.000000 |
   | ff3710b8-f110-4949-b578-b09a1dbc19bb | nova-scheduler | nova-scheduler-5bcc764f79-5hcvx | internal | enabled | up    | 2023-05-31T12:16:21.000000 |
   | d6831741-677e-471f-a019-66b46150cbcc | nova-scheduler | nova-scheduler-5bcc764f79-sfclc | internal | enabled | up    | 2023-05-31T12:16:20.000000 |
   | 792ec442-5e04-4a5f-9646-7cb0001dfb9c | nova-conductor | nova-conductor-56dfd9749-s5c6j  | internal | enabled | up    | 2023-05-31T12:16:21.000000 |
   | 848f1573-3706-49ab-8c57-d6edf1631dce | nova-conductor | nova-conductor-56dfd9749-dfkgd  | internal | enabled | up    | 2023-05-31T12:16:21.000000 |
   | c5217922-bc1d-446e-a951-a4871d6020e3 | nova-compute   | compute2                        | nova     | enabled | up    | 2023-05-31T12:16:25.000000 |
   | 5f8cbde0-3c5f-404c-b31e-da443c1f14fd | nova-compute   | compute1                        | nova     | enabled | up    | 2023-05-31T12:16:25.000000 |
   +--------------------------------------+----------------+---------------------------------+----------+---------+-------+----------------------------+

* 모든 서비스가 활성화되어 있어야 합니다.
* 각 compute 노드에는 nova-compute 서비스가 있어야 합니다.



netapp,ceph 순서 확인
+++++++++++++++++++++++


cinder volume pod 접속합니다.

::

   root@btx-0:/# k get po -l component=volume
   NAME                            READY   STATUS    RESTARTS   AGE
   cinder-volume-98c8fbff6-jsrzx   1/1     Running   0          14h
   cinder-volume-98c8fbff6-spr5x   1/1     Running   0          14h
   cinder-volume-98c8fbff6-xvw8n   1/1     Running   0          14h


특정 pod(cinder-volume-98c8fbff6-jsrzx)에 접속합니다.

::

   root@btx-0:/# k exec -it cinder-volume-98c8fbff6-jsrzx -c cinder-volume -- bash


cinder.conf에서 default_volume_type와 enabled_backends 항목을 찾습니다.

::

   cinder@cinder-volume-98c8fbff6-jsrzx:/etc/cinder$ grep -E 'default_volume_type|enabled_backends' cinder.conf
   default_volume_type = rbd1
   enabled_backends = rbd1,netapp1


.. _test-section:


Test
++++++

The command "btx --test"

* provider 네트워크와 서브넷을 생성합니다.
* provider 네트워크를 생성할 때 주소 pool 범위를 입력합니다.
* cirros 이미지를 생성합니다.
* 보안 그룹 규칙을 추가합니다.
* flavor를 생성합니다.
* instance를 생성합니다.
* 볼륨을 생성합니다.
* 볼륨을 instance에 연결합니다.

모든 것이 잘 진행되면 출력은 다음과 같습니다.

::

   $ btx --test
   ...
   Creating provider network...
   Type the provider network address (e.g. 192.168.22.0/24): 192.168.22.0/24
   Okay. I got the provider network address: 192.168.22.0/24
   The first IP address to allocate (e.g. 192.168.22.100): 192.168.22.100
   Okay. I got the first address in the pool: 192.168.22.100
   The last IP address to allocate (e.g. 192.168.22.200): 192.168.22.108
   Okay. I got the last address of provider network pool: 192.168.22.108
   ...
   Instance status
   +------------------+------------------------------------------------------------------------------------+
   | Field            | Value                                                                              |
   +------------------+------------------------------------------------------------------------------------+
   | addresses        | public-net=192.168.22.104                                                          |
   | flavor           | disk='1', ephemeral='0', , original_name='m1.tiny', ram='512', swap='0', vcpus='1' |
   | image            | cirros (0b2787c1-fdb3-4a3c-ba9d-80208346a85c)                                      |
   | name             | test                                                                               |
   | status           | ACTIVE                                                                             |
   | volumes_attached | delete_on_termination='False', id='76edcae9-4b17-4081-8a23-26e4ad13787f'           |
   +------------------+------------------------------------------------------------------------------------+

provider 네트워크가 연결된 서버에서 ssh를 사용하여 provider 네트워크 IP를 통해 instance에 연결합니다.

::

   (a node on provider network)$ ssh cirros@192.168.22.104
   cirros@192.168.22.104's password:
   $ ip address show dev eth0
   2: eth0:<BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1450 qdisc pfifo_fast qlen 1000
       link/ether fa:16:3e:ed:bc:7b brd ff:ff:ff:ff:ff:ff
       inet 192.168.22.104/24 brd 192.168.22.255 scope global eth0
          valid_lft forever preferred_lft forever
       inet6 fe80::f816:3eff:feed:bc7b/64 scope link
          valid_lft forever preferred_lft forever

암호는 설정된 cirros 암호입니다.

(힌트: 비밀번호는 시카고 컵스 야구팀을 사랑하는 누군가가 만든 것 같습니다.)




