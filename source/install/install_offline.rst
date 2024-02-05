=======================================
Burrito 오프라인 설치가이드
=======================================


.. This content will be ignored during compilation
   .. contents::
      :local:
      :backlinks: none
      :depth: 2

오프라인 환경에서 Burrito를 설치하는 가이드입니다.

Burrito ISO를 사용하여 설치합니다.

burrito_iso 프로젝트를 사용하여 직접 Burrito_iso를 만들수 있습니다.


지원 운영체제
---------------

* Rocky Linux 8.x

시스템 사양
-------------

Burrito를 설치하기 위한 최소 시스템 요구사양입니다.

========= ========= =========== ========= ===========
node role CPU (ea)  Memory (GB) Disk (GB) Extra Disks
--------- --------- ----------- --------- -----------
control   8         16          50        N/A        
--------- --------- ----------- --------- -----------
compute   4         8           50        N/A        
--------- --------- ----------- --------- -----------                            
storage   4         8           50        3 ea x 50GB        
========  ========= =========== ========= =========== 

리소스가 더 많은 경우, 각 노드에 더 많은 자원이 필요합니다.

네트워크
-----------

Burrito에서 기본적으로 사용하는 네트워크는 총 5개입니다.

* service network: 웹 포탈 서비스 / 서버 접속 용도 네트워크 (예: 192.168.20.0/24)
* management network: K8S, Openstack 관리 네트워크 (예: 192.168.21.0/24)
* provider network: 가상 PC에 할당할 네트워크 (예: 192.168.22.0/24)
* overlay network: OpenStack 오버레이 네트워크 (예: 192.168.23.0/24)
* storage network: Ceph Public(Mon)/Cluster(OSD) 네트워크 (예: 192.168.24.0/24)

각 네트워크의 용도를 모르는 경우, openstack 전문가와 상의하세요.

네트워크 구조 예시
++++++++++++++++++++

아래는 네트워크 설계 가이드 예시입니다. 

* control/compute 머신은 5개의 네트워크를 모두 가지고 있습니다.
* provider network는 네트워크에는 IP 주소가 할당되지 않습니다.
* storage 머신은 2개의 네트워크를 가지고 있습니다. (management and storage)

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

* KeepAlived VIP on management: 192.168.21.100
* KeepAlived VIP on service: 192.168.20.100

전제 조건
----------

* 운영체제는 Burrito ISO를 사용하여 설치합니다.
* control 그룹의 첫번째 노드는 서버를 구성하는 노드입니다.
* 모든 노드의 Ansible 사용자는 sudo 권한을 가지고 있습니다. 이 문서의 ansible 유저는 clex 입니다.
* 모든 노드는 배포 노드의 /etc/hosts 에 있어야 합니다.

아래는 배포 노드의 /etc/hosts 예시입니다.

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
-------

iso 파일을 사용 가능하게 연결한다.()

::

   $ sudo mount -o loop,ro <path/to/burrito_iso_file> /mnt

/mnt 내부에 burrito를 체크한다.

::

   $ ls /mnt/burrito-*.tar.gz
   /mnt/burrito-<version>.tar.gz

burrito 압축 파일을 홈 디렉토리에 압축 해제한다.

::

   $ tar xzf /mnt/burrito-<version>.tar.gz

burrito 디렉토리로 들어간다.

::

   $ cd burrito-<version>

오프라인을 인자로 prepare.sh 스크립트를 실행한다.

::

   $ ./prepare.sh offline
   Enter management network interface name: eth1

매니지먼트 네트워크 인터페이스 이름을 입력하라는 메세지가 표시됩니다.
매니지먼트 네트워크 이름을 입력한다. (예: eth1)

인벤토리 호스트 및 변수
++++++++++++++++++++++++++++

burrito에는 4개의 호스트 그룹이 있습니다.

* 컨트롤 노드(Control Node): 쿠버네티스 및 오픈스택 제어 플레인 구성요소를 실행
* 네트워크 노드(Network Node): 쿠버네티스 워커 및 오픈스택 네트워크 서비스를 실행
* 컴퓨팅 노드(Compute Node): 쿠버네티스 워커 및 오픈스택 하이퍼바이저와 인스턴스를 운영하기 위한
 네트워크 에이전트를 실행
* 스토리지 노드(Storage Node): Ceph 스토리지 서비스인 monitor, manager, osd, rados gateway를 실행

네트워크 노드는 선택사항입니다. 일반적으로 컨트롤 노드는 컨트롤 노드와 네트워크 노드 역할을
모두 수행한다.

인벤토리 호스트 편집
^^^^^^^^^^^^^^^^^^^^^^^

다음은 샘플 인벤토리 파일입니다.

* hosts.sample (default):
    이 파일은 storage 백엔드 ceph를 사용하는 샘플 파일입니다.
* hosts_powerflex.sample:
    이 파일은 storage 백엔드 powerflex를 사용하는 샘플 파일입니다.
* hosts_powerflex_hci.sample:
    이 파일은 powerflex HCI(Hyper-Converged Infrastructure)를 사용하는 샘플 파일입니다.
* hosts_hitachi.sample:
    이 파일은 hitachi를 스토리지 벡엔드로 사용하는 샘플 파일입니다.

    
.. 경고::
    powerflex를 burrito에 설치하려면 Dell에서 powerflex rpm 패키지를 지원받아야 합니다.

.. 경고::
    HSPC(hitachi storage Plug-in for Containers) 이미지를 burrito에 설치하려면 Hitachi 에서
    hitachi 컨테이너 이미지를 받아야 합니다.

prepare.sh 스크립트를 실행하면 기본 hosts.sample이 hosts 파일로 복사됩니다.

powerflex 스토리지를 사용하려면 powerflex 인벤토리 파일중 하나를 복사하세요.::

   $ cp hosts_powerflex_hci.sample hosts

hitachi 스토리지를 사용하려면 hitachi 인벤토리 파일을 복사하세요.::

   $ cp hosts_hitachi.sample hosts

아래 샘플 인벤토리 파일들이 있습니다.

.. collapse:: 기본 인벤토리 파일

   .. code-block::
      :linenos:

      control1 ip=192.168.21.101 ansible_connection=local ansible_python_interpreter=/usr/bin/python3
      control2 ip=192.168.21.102
      control3 ip=192.168.21.103
      compute1 ip=192.168.21.104
      compute2 ip=192.168.21.105
      storage1 ip=192.168.21.106
      storage2 ip=192.168.21.107
      storage3 ip=192.168.21.108

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

.. collapse:: powerflex 인벤토리 파일

   .. code-block::
      :linenos:

      control1 ip=192.168.21.101 ansible_connection=local ansible_python_interpreter=/usr/bin/python3
      control2 ip=192.168.21.102
      control3 ip=192.168.21.103
      compute1 ip=192.168.21.104
      compute2 ip=192.168.21.105
      storage1 ip=192.168.21.106
      storage2 ip=192.168.21.107
      storage3 ip=192.168.21.108

      # ceph nodes
      [mons]
      [mgrs]
      [osds]
      [rgws]
      [clients]

      # powerflex nodes
      [mdm]
      storage[1:3]

      [sds]
      storage[1:3]

      [sdc]
      control[1:3]
      compute[1:2]

      [gateway]
      storage[1:2]

      [presentation]
      storage3

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

.. collapse:: powerflex HCI 인벤토리 파일

   .. code-block::
      :linenos:

      pfx-1 ip=192.168.21.131 ansible_connection=local ansible_python_interpreter=/usr/bin/python3
      pfx-2 ip=192.168.21.132
      pfx-3 ip=192.168.21.133

      # ceph nodes
      [mons]
      [mgrs]
      [osds]
      [rgws]
      [clients]

      # powerflex nodes
      [mdm]
      pfx-[1:3]

      [sds]
      pfx-[1:3]

      [sdc]
      pfx-[1:3]

      [gateway]
      pfx-[1:2]

      [presentation]
      pfx-3

      # kubernetes nodes
      [kube_control_plane]
      pfx-[1:3]

      [kube_node]
      pfx-[1:3]

      # openstack nodes
      [controller-node]
      pfx-[1:3]

      [network-node]
      pfx-[1:3]

      [compute-node]
      pfx-[1:3]

      ###################################################
      ## Do not touch below if you are not an expert!!! #
      ###################################################

.. collapse:: hitachi 인벤토리 파일

   .. code-block::
      :linenos:

      control1 ip=192.168.21.101 ansible_connection=local ansible_python_interpreter=/usr/bin/python3
      control2 ip=192.168.21.102
      control3 ip=192.168.21.103
      compute1 ip=192.168.21.104
      compute2 ip=192.168.21.105
      storage1 ip=192.168.21.106
      storage2 ip=192.168.21.107
      storage3 ip=192.168.21.108
      
      # ceph nodes
      [mons]
      [mgrs]
      [osds]
      [rgws]
      [clients]
      
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


.. 경고::
   이 샘플 파일에는 네트워크 노드가 없으므로 컨트롤 노드가 네트워크 그룹에
   있음을 주의하세요.

vars.yml 편집합니다.
^^^^^^^^^^^^^^^^^^^^^

.. code-block:: yaml
   :linenos:

   ---
   ### 네트워크 인터페이스명 정의.
   # overlay 네트워크를 설정하지 않으려면 overlay_iface_name을 null로 설정하세요.
   # 그후, provider 네트워크만 설정합니다.
   svc_iface_name: eth0
   mgmt_iface_name: eth1
   provider_iface_name: eth2
   overlay_iface_name: eth3
   storage_iface_name: eth4
   
   ### ntp
   # 컨트롤 노드의 대한 시간 서버를 지정해야 합니다.
   # 네트워크에 기본 ntp.org 서버 또는 시간 서버를 사용할 수 있습니다.
   # 만약 서버가 오프라인이고 네트워크에 시간 서버가 없으면,
   #   ntp_servers를 빈 목록으로 설정하세요.  
   #   그런 다음 컨트롤 노드는 다른 노드의 ntp 서버가 됩니다.
   # ntp_servers: []
   ntp_servers:
     - 0.pool.ntp.org
     - 1.pool.ntp.org
     - 2.pool.ntp.org
   
   ### keepalived VIP 매니지먼트 네트워크 연결(필수)
   keepalived_vip: ""
   # keepalived VIP 서비스 네트워크 연결 (선택)
   # 메니지먼트 네트워크에 직접 엑세스할 수 없는 경우 설정하세요.
   # 서비스 네트워크를 통해 horizon 대쉬보드에 엑세스해야 합니다..
   keepalived_vip_svc: ""
   
   ### metallb
   # metallb LoadBalancer를 사용하려면, true로 설정합니다.
   metallb_enabled: false
   # MetalLB LoadBalancer IP 범위 또는 cidr 표기법을 설정합니다.
   # IP 범위: 192.168.20.95-192.168.20.98 (4개의 ip할당 가능)
   # CIDR: 192.168.20.128/26 (192.168.20.128 - 191 지정가능.)
   # 하나의 IP: 192.168.20.95/32
   metallb_ip_range:
     - "192.168.20.95-192.168.20.98"
   
   ### HA tuning
   # ha 단계: moderato, allegro, and vivace
   # moderato: 기본 라이브니스 업데이트 및 failover 응답
   # allegro: 더 빠른 라이브니스 업데이트 및 failover 응답
   # vivace: 가장 빠른 라이브니스 업데이트 및 failover 응답
   ha_level: "moderato"
   k8s_ha_level: "moderato"
   
   ### 스토리지
   # 스토리지 백엔드: ceph 또는 netapp
   # 여러 백엔드가 있는경우, 첫번째 백엔드가 기본 백엔드입니다..
   storage_backends:
     - ceph
     - netapp
     - powerflex
     - hitachi
   
   # ceph: ceph 설정은 group_vars/all/ceph_vars.yml 에 있습니다.
   # netapp: netapp 설정은 group_vars/all/netapp_vars.yml 에 있습니다.
   # powerflex: powerflex 설정은 group_vars/all/powerflex_vars.yml 에 있습니다.
   # hitachi: hitachi 설정은 group_vars/all/hitachi_vars.yml 에 있습니다.

   ##############################################
   ## 전문가가 아니라면 아래를 편집하지 마세요!!!  #
   ##############################################

각 변수의 설명
^^^^^^^^^^^^^^^^

\*_iface_name
  각 네트워크 인터페이스 이름을 설정하세요.

  provider 네트워크만 설정하려면 overlay_iface_name을 null로 설정하세요.
  그후, openstack neutron은 자체 서비스(overlay) 네트워크를 비활성화합니다.

ntp_servers (default: {0,1,2}.pool.ntp.org)
  컨트롤 노드의 시간 서버를 지정하세요.
  기본 ntp.org 서버 또는 네트워크의 시간 서버를 사용할 수 있습니다.

  서버가 오프라인 상태이고 네트워크에 시간서버가 없는 경우,
  ntp_servers를 빈리스트로 설정하세요.(ntp_servers: []). 그런 다음 제어노드는
  다른 노드의 NTP 서버가 됩니다.

keepalived_vip (mandatory)
  내부 서비스에 대한 로드 밸런싱 및 고가용성을 위해 management 네트워크에 VIP
  주소를 할당하세요. 이는 필수입니다.

keepalived_vip_svc (optional)
  horizon 대시보드 서비스를 위해 서비스 네트워크에 VIP주소를 할당하세요.
  관리 네트워크에 직접 액세스할 수 없는 경우 이를 설정하세요.

  할당하지 않으면 관리 네트워크의 keepalived_vip를 통해 horizon
  대시보드에 연결해야 합니다.

metallb_enabled (default: false)
  metallb LoadBalancer를 사용하려면 true로 설정하세요.
  ( ` metallb는 무엇인가? <https://metallb.universe.tf/>`_)

metallb_ip_range
  metallb LoadBalancer IP 범위 또는 cidr 표기법을 설정하세요.

  * IP 범위: 192.168.20.95-192.168.20.98 (4 IP를 할당 가능하다.)
  * CIDR: 192.168.20.128/26 (192.168.20.128 - 191 할당 가능하다.)
  * Only one IP: 192.168.20.95/32 (192.168.20.95 할당 가능하다.)

ha_level
  KeepAlived/HAProxy HA 설정합니다..
  moderato(기본값), allegro, vivace 중 하나를 선택할 수 있습니다.
  각 레벨은 다음 매개변수를 설정합니다..

  * interval: health check 초 단위 간격
  * timeout: health check 초 단위 타임아웃
  * rise: 요구된 성공 횟수
  * fall: 요구된 실패 횟수

k8s_ha_level
  쿠버네티스 HA 레벨을 설정합니다.
  moderato(default), allegro vivace 중 하나를 선택할 수 있습니다.
  각 레벨은 다음 매개변수를 설정합니다.

  * node_status_update_frequency: 
    kubelet이 마스터 노드 상태를 게시하는 빈도를 지정합니다.
  * node_monitor_period:
    NodeController에서 NodeStatus를 동기화하는 주기입니다.
  * node_monitor_grace_period:
    실행 중인 노드가 응답하지 않는 것으로 간주하기 전에 허용되는 시간입니다.
  * not_ready_toleration_seconds:
    이미 해당 관용구를 가지고 있지 않은 모든 포드에 기본적으로 추가되는 notReady:NoExecute 관용구의 관용 기간입니다.
  * unreachable_toleration_seconds:
    도달할 수 없는:NoExecute 관용구에 기본적으로 추가되는 관용구의 관용 기간입니다??
  * kubelet_shutdown_grace_period:
    노드가 종료를 지연해야 하는 총 시간입니다.
  * kubelet_shutdown_grace_period_critical_pods:
    노드 종료시 중요한 파드를 종료하는데 사용되는 주기입니다.

storage_backends
  Burrito는 다음과 같은 스토리지 백엔드를 지원합니다 -
  ceph, netapp, powerflex, and hitachi.

  여러 백엔드가 있는 경우 첫번째 백엔드가 기본 백엔드입니다.
  즉, 기본 storageclass, glance 저장소 및 기본 cinder 볼륨 유형은 첫번째 백엔드입니다.
  
  k8s의 영구 볼륨은 storagecless 이름을 지정하지 않으면 기본 백엔드에 생성됩니다.
  
  오픈스택의 볼륨은 볼륨 유형을 지정하지 않으면 기본 백엔드에 생성됩니다.

저장 변수
++++++++++

ceph
^^^^^

만약 스토리지 백엔드가 ceph인 경우, 
스토리지 노드에서 lsblk 명령어를 실행하여 장치이름을 확인한다.

.. code-block:: shell

   storage1$ lsblk -p
   NAME        MAJ:MIN RM SIZE RO TYPE MOUNTPOINT
   /dev/sda      8:0    0  50G  0 disk 
   └─/dev/sda1   8:1    0  50G  0 part /
   /dev/sdb      8:16   0  50G  0 disk 
   /dev/sdc      8:32   0  50G  0 disk 
   /dev/sdd      8:48   0  50G  0 disk 

이 경우, /dev/sda 가 운영체제 디스크이고 /dev/sd{b,c,d}는
ceph OSD 디스크입니다.

group_vars/all/ceph_vars.yml 을 편집하세요.

.. code-block::
   :linenos:

   ---
   # ceph config
   lvm_volumes:
     - data: /dev/sdb
     - data: /dev/sdc
     - data: /dev/sdd
   ...

netapp
^^^^^^^

netapp 스토리지 백엔드의 경우, group_vars/all/netapp_vars.yml을 편집하세요.

.. code-block::
   :linenos:

   ---
   netapp:
     - name: netapp1
       managementLIF: "192.168.100.230"
       dataLIF: "192.168.140.19"
       svm: "svm01"
       username: "admin"
       password: "<netapp_admin_password>"
       nfsMountOptions: "lookupcache=pos"
       shares:
         - /dev03
   ...

특정 NFS 버전을 사용하려면 nfsMountOption의 nfsvers를 추가할 수 있습니다.

예를들어, nfs version 4.0을 사용하려면 nfsMountOptions에 nfsvers=4.0을 입력하세요.
(nfsMountOptions: "nfsvers=4.0,lookupcache=pos")
그런 다음, NetApp NFS 스토리지에서 NFS 버전 4가 활성화되어 있는지 확인해야 합니다.

만약 이러한 변수들이 무엇을 의미하는지 모르겠다면, Netapp 엔지니어에게 문의하세요.

powerflex
^^^^^^^^^^

powerflex 스토리지 백엔드의 경우,
스토리지 노드에서 lsblk 명령어를 입력해서 디바이스명을 확인한다.

.. code-block::
   :linenos:

   storage1$ lsblk -p
   NAME        MAJ:MIN RM SIZE RO TYPE MOUNTPOINT
   /dev/sda      8:0    0  50G  0 disk
   └─/dev/sda1   8:1    0  50G  0 part /
   /dev/sdb      8:16   0  50G  0 disk
   /dev/sdc      8:32   0  50G  0 disk
   /dev/sdd      8:48   0  50G  0 disk

이 경우, /dev/sda 는 운영체제 디스크이고 /dev/sd{b,c,d} 는 powerflex SDS 디스크이다.

group_vars/all/powerflex_vars.yml 을 편집합니다.

.. code-block::
   :linenos:

   # MDM VIPs on storage networks
   mdm_ip:
     - "192.168.24.100"
   storage_iface_names:
     - eth4
   sds_devices:
     - /dev/sdb
     - /dev/sdc
     - /dev/sdd

   #
   # Do Not Edit below
   #

만약 이 변수들이 무엇을 의미하는지 모르겠다면, 
Dell 엔지니어에게 문의하세요.

hitachi
^^^^^^^

burrito에서 히타치 스토리지를 사용하기 전에,
호스트 그룹 및 호스트 모드 옵션을 수동으로 설정해야 합니다.

참고바랍니다.
:문서:`히타치 스토리지 매뉴얼 설정 가이드 <setup_hitachi_storage>`.

히타치 스토리지 백엔드의 경우, group_vars/all/hitachi_vars.yml를 편집하세요.

.. code-block::
   :linenos:

   ---
   # 스토리지 모델: 저장소 모델 hitachi_prefix_id를 아래에서 확인하세요.
   hitachi_storage_model: vsp_e990
   
   ## k8s 스토리지클래스 변수들
   # 히타치 스토리지 시리얼 넘버를 확인한다.
   hitachi_serial_number: "<serial_number>"
   hitachi_pool_id: "0"
   # k8s PV에서 사용할 포트 식별자
   hitachi_port_id: "CL4-A"
   
   ## openstack cinder 변수들
   hitachi_san_ip: "<san_ip>"
   hitachi_san_login: "<san_login>"
   hitachi_san_password: "<san_password>"
   hitachi_ldev_range: "00:10:00-00:10:FF"
   hitachi_target_ports: "CL3-A"
   hitachi_compute_target_ports: "CL1-A,CL2-A,CL3-A,CL5-A,CL6-A"
   
   ###########################
   # 아래를 수정하지 마세요!!! #
   ###########################

저장소에 관한 정보를 얻으려면 히타치 엔지니어에게 문의하세요.

* hitachi_storage_model: hitachi_prefix_id 변수 값 중 하나를 입력하세요.
* hitachi_serial_number: 6자리 시리얼 번호
* hitachi_pool_id: 히타치 스토리지 풀 ID
* hitachi_port_id: 쿠버네티스 포트 ID
* hitachi_san_ip: 히타치 컨트롤러의 IP주소
* hitachi_san_login: 히타치 컨트롤러의 사용자 이름
* hitachi_san_password: 히타치 컨트롤러의 비밀번호
* hitachi_ldev_range: 씬더 드라이버에서 사용할 수 있는 LDEV 번호 범위를 
  ‘aa:bb:cc-dd:ee:ff’ 형식으로 입력하세요.
* hitachi_target_ports: 볼륨을 제어노드에 연결하는데 사용되는 저장소 포트ID
* hitachi_compute_target_ports: 제어 및 컴퓨팅 노드에 볼륨을 연결하는데 사용되는
저장소 포트ID

vault secret 파일 생성
+++++++++++++++++++++++++++

비밀번호를 암호화하기 위해 vault 파일을 생성합니다.::

   $ ./run.sh vault
   <user> password:
   openstack admin password:
   Encryption successful

다른 노드에 ssh 연결을 위한 사용자 비밀번호를 입력하세요.

오픈스택 horizon 대시보드에 연결할 때 사용할 오픈스택 관리자 비밀번호를
입력하세요.

연결 확인
+++++++++++++

다른 노드 연결을 확인하세요.::

   $ ./run.sh ping

모든 노드에서 성공을 확인해야 합니다.

설치
--------

각 플레이북 실행 시 PLAY REACAP에서 실패한 작업이 없어야 합니다.

예를 들어::

   PLAY RECAP *****************************************************************
   control1                   : ok=20   changed=8    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0   
   control2                   : ok=19   changed=8    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0   
   control3                   : ok=19   changed=8    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0   

각 단계에는 결과 프로세스가 있으며, 다음 단계로 넘어가기 전에 확인해주세요.

.. 경고::
   **결과 확인시 실패작업이 있을경우 다음단계를 진행하지 마세요.**

Step.1 Preflight
+++++++++++++++++

Preflight 설치 단계는 다음 작업을 구현합니다.

* 로컬 yum 리포지토리를 설정합니다.
* NTP 타임 서버 및 클라이언트 구성.
* 공개 ssh 키를 다른 노드에 배포합니다. (deploy_ssh_key 가 true 인 경우).

설치
^^^^^^^

preflight 플레이북 실행.::

   $ ./run.sh preflight

확인
^^^^^

로컬 yum 리포지토리가 모든 노드에 설정 되어있는지 확인합니다.::

   $ sudo dnf repolist
   repo id                               repo name
   burrito                               Burrito Repo

ntp 서버 및 클라이언트가 구성되어 있는지 확인한다.

ntp_servers를 빈 목록(ntp_servers: [])으로 설정하면,
각 제어 노드는 다른 제어 노드를 타임 서버로 가져야 한다.::

   control1$ chronyc sources
   MS Name/IP address      Stratum Poll Reach LastRx Last sample               
   ========================================================================
   ^? control2             9   6   377   491   +397ms[ +397ms] +/-  382us
   ^? control3             9   6   377   490   -409ms[ -409ms] +/-  215us

컴퓨트/스토리지 노드에는 시간 서버로 제어 노드가 있어야 한다.::

   $ chronyc sources
   MS Name/IP address      Stratum Poll Reach LastRx Last sample               
   ========================================================================
   ^* control1             8   6   377    46    -15us[  -44us] +/-  212us
   ^- control2             9   6   377    47    -57us[  -86us] +/-  513us
   ^- control3             9   6   377    47    -97us[ -126us] +/-  674us

Step.2 HA 
++++++++++

HA 설치 단계는 다음과 같은 작업을 구현한다.

* KeepAlived 서비스를 설정합니다..
* HAProxy 서비스를 설정합니다.

KeepAlived 및 HAProxy 서비스는 burrito 플랫폼의 필수 서비스입니다.

로컬 컨테이너 레지스트리, 로컬 yum 리포지토리,
Ceph Rados Gateway 서비스는 해당 서비스에 종속성을 갖는다.

설치
^^^^^

HA 스택 플레이북 실행.::

   $ ./run.sh ha

확인
^^^^^^

컨트롤 노드에서 keepalived 및 haproxy가 실행중인지 확인한다.::

   $ sudo systemctl status keepalived haproxy
   keepalived.service - LVS and VRRP High Availability Monitor
   ...
      Active: active (running) since Wed 2023-05-31 17:29:05 KST; 6min ago
   ...
   haproxy.service - HAProxy Load Balancer
   ...
      Active: active (running) since Wed 2023-05-31 17:28:52 KST; 8min ago

management 인터페이스에 keepalived_vip가 생성되었는지 확인합니다 
첫번째 컨트롤 노드.::

   $ ip -br -4 address show dev eth1
   eth1             UP             192.168.21.101/24 192.168.21.100/32 

서비스 인터페이스에 keepalived_vip_svc가 생성되었는지 확인합니다 
설정을 했을 경우 첫번째 컨트롤 노드.::

   $ ip -br -4 address show dev eth0
   eth0             UP             192.168.20.101/24 192.168.20.100/32 

Step.3 Ceph
+++++++++++

스토리지 백엔드로 ceph를 정의하지 않았으면 이 단계를 건너 뛰세요.

Ceph 설치 단계는 다음과 같은 작업을 구현합니다.

* 스토리지 노드에 ceph 서버 및 클라이언트 패키지를 설치합니다.
* 다른 노드에 ceph 클라이언트 패키지를 설치합니다.
* 스토리지 노드에 ceph monitor, manager, osd, rados gateway 서비스를
   설정합니다.

설치
^^^^^

ceph가 스토리지 백엔드에 있으면 ceph 플레이북을 실행합니다.::

   $ ./run.sh ceph

확인
^^^^^^

ceph 플레이북 실행 후 ceph 상태 확인.::

   $ sudo ceph health
   HEALTH_OK

HEALTH_OK 확인한다.

자세한 상태를 확인하려면, `sudo ceph -s` 명령어를 실행한다.
아래와 같이 출력된다.::

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

4가지 서비스가 있습니다. - mon, mgr, osd, and rgw.

때때로 Health_WARN <something> 과 함께 recently crashed 로 표시될수 있습니다.
괜찮아요. 대부분 무해한 경고입니다.

충돌 목록을 나열합니다.::

   $ sudo ceph crash ls

모든 충돌 기록입니다.::

   $ sudo ceph crash archive-all

이후, ceph 상태를 다시 확인합니다. HEALTH_OK가 나옵니다.

Step.4 Kubernetes
+++++++++++++++++

쿠버네티스 설치 단계는 다음과 같은 작업을 구현한다.

* 쿠버네티스 노드에 쿠버네티스 이진파일을 설치합니다.
* 쿠버네티스 컨트롤 플레인을 설정합니다.
* 쿠버네티스 워커 노드를 설정합니다.
* kube-system 네임스페이스에 로컬 레지스트리를 설정합니다.

설치
^^^^^^^

k8s 플레이북을 실행.::

   $ ./run.sh k8s

확인
^^^^^^

모든 노드가 Ready 상태인지 확인한다.::

   $ sudo kubectl get nodes
   NAME       STATUS   ROLES           AGE   VERSION
   compute1   Ready    <none>          15m   v1.28.3
   compute2   Ready    <none>          15m   v1.28.3
   control1   Ready    control-plane   17m   v1.28.3
   control2   Ready    control-plane   16m   v1.28.3
   control3   Ready    control-plane   16m   v1.28.3


Step.5.1 Netapp
++++++++++++++++

스토리지 백엔드로 netapp 정의하지 않았으면 이 단계를 건너 뛰세요.

Netapp 설치 단계는 다음과 같은 작업을 구현한다.

* 트라이던트 네임스페이스에 트라이던트 구성요소를 설치합니다.
* netapp 백엔드를 설정합니다.
* netapp 스토리지 클래스를 만듭니다.

설치
^^^^^^^

netapp 플레이북 실행.::

   $ ./run.sh netapp

확인
^^^^^

모든 파드가 Running 이고 Ready가 되었는지 확인하세요.::

   $ sudo kubectl get pods -n trident
   NAME                           READY   STATUS    RESTARTS   AGE
   trident-csi-6b96bb4f87-tw22r   6/6     Running   0          43s
   trident-csi-84g2x              2/2     Running   0          42s
   trident-csi-f6m8w              2/2     Running   0          42s
   trident-csi-klj7h              2/2     Running   0          42s
   trident-csi-kv9mw              2/2     Running   0          42s
   trident-csi-r8gqv              2/2     Running   0          43s

netapp 스토리지 클래스가 생성되었는지 확인.::

   $ sudo kubectl get storageclass netapp
   NAME               PROVISIONER             RECLAIMPOLICY   VOLUMEBINDINGMODE   ALLOWVOLUMEEXPANSION   AGE
   netapp (default)   csi.trident.netapp.io   Delete          Immediate           true                   20h

Step.5.2 Powerflex
+++++++++++++++++++

스토리지 백엔드로 powerflex를 정의하지 않았으면 이 단계를 건너 뛰세요.

powerflex 설치 단계는 다음과 같은 작업을 구현한다.

* powerflex rpm 패키지를 설치합니다.
* powerflex MDM cluster를 생성합니다.
* gateway 및 프리젠테이션 서비스를 구성합니다.
* Protection Domain, Storage Pool, 및 SDS 디바이스를 설정합니다.
* vxflexos 네임스페이스에 vxflexos 컨트롤러 및 노드를 설치합니다.
* powerflex 스토리지 클래스를 생성합니다.

설치
^^^^^

powerflex 플레이북을 실행.::

   $ ./run.sh powerflex

확인
^^^^^

vxflexos 네임스페이스의 모든 파드가 실행중이고 준비 상태인지 확인.::

   $ sudo kubectl get pods -n vxflexos
   NAME                                   READY   STATUS    RESTARTS   AGE
   vxflexos-controller-744989794d-92bvf   5/5     Running   0          18h
   vxflexos-controller-744989794d-gblz2   5/5     Running   0          18h
   vxflexos-node-dh55h                    2/2     Running   0          18h
   vxflexos-node-k7kpb                    2/2     Running   0          18h
   vxflexos-node-tk7hd                    2/2     Running   0          18h

powerflex 스토리지 클래스가 생성되었는지 확인.::

   $ sudo kubectl get storageclass powerflex
   NAME                  PROVISIONER                RECLAIMPOLICY   VOLUMEBINDINGMODE      ALLOWVOLUMEEXPANSION   AGE
   powerflex (default)   csi-vxflexos.dellemc.com   Delete          WaitForFirstConsumer   true                   20h

Step.5.3 Hitachi
+++++++++++++++++

스토리지 백엔드로 hitachi를 정의하지 않았으면 이 단계를 건너 뛰세요.

Hitachi설치 단계는 다음과 같은 작업을 구현한다.

* hspc-operator-system 네임스페이스에 HSPC(Hitachi Storage Plug-in for Containers)를 설치합니다.
* Hitachi 스토리지 클래스를 생성합니다.

설치
^^^^^

hitachi 플레이북 실행.::

   $ ./run.sh hitachi

확인
^^^^^^

hspc-operator-system 네임스페이스에서 모든 Pod가 실행 중이고 준비되었는지 확인.::

   $ sudo kubectl get pods -n hspc-operator-system
   NAME                                                READY   STATUS    RESTARTS        AGE
   hspc-csi-controller-7c4cbdccbc-sh7lz                6/6     Running   0               40s
   hspc-csi-node-2snpm                                 2/2     Running   0               42s
   hspc-csi-node-2t897                                 2/2     Running   0               42s
   hspc-csi-node-xd78f                                 2/2     Running   0               42s
   hspc-operator-controller-manager-599b69557b-6v9k7   1/1     Running   0               35s

powerflex storageclass 가 생성되었는지 확인.::

   $ sudo kubectl get storageclass hitachi
   NAME                PROVISIONER            RECLAIMPOLICY   VOLUMEBINDINGMODE   ALLOWVOLUMEEXPANSION   AGE
   hitachi (default)   hspc.csi.hitachi.com   Delete          Immediate           true                   30s

Step.6 Patch
+++++++++++++

patch 설치 단계는 다음 작업을 수행합니다.

* ceph가 storage_backends에 있으면 ceph-csi 드라이버를 설치합니다..
* kube-apiserver 를 패치합니다.

설치
^^^^^^^

patch 플레이북 실행.::

   $ ./run.sh patch

확인
^^^^^

패치 후 kube-apiserver를 다시 시작하는데 시간이 걸립니다.

모든 파드가 Running 상태를 확인하고 kube-system 네임스페이스에 Running 상태인지 확인.

.. collapse:: kube-system 네임스페이스 파드 목록

   .. code-block:: shell

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

레지스트리 파드가 실행되고 준비될 때까지 기다립니다.


Step.7 Registry
+++++++++++++++

Registry 설치 단계는 다음 작업을 구현합니다.

* registry 파드 이름을 가져옵니다.
* ISO에서 registry 파드로 컨테이너 이미지를 복사합니다.

설치
^^^^^^^

registry 플레이북을 실행.::

   $ ./run.sh registry

확인
^^^^^^

이미지가 로컬 registry에 있는지 확인합니다.::

   $ curl -s <keepalived_vip>:32680/v2/_catalog | jq
   {
       "repositories": [
           "airshipit/kubernetes-entrypoint",
           "calico/cni",
           "calico/kube-controllers",
           ...
           "sig-storage/csi-resizer",
           "sig-storage/csi-snapshotter"
       ]
   }

출력의 리포지토리는 비워둘 수 없습니다.

Step.8 Landing
+++++++++++++++

Landing 설치 단계에서는 다음 작업을 수행 합니다.

* 컨트롤 노드에 제네시스 레지스트리 서비스를 배포합니다.
* 패치 부트스트랩 파드 (kube-{apiserver,scheduler,controller-manager},
  kube-proxy, local registry 및 csi driver pods) 를 통해 이미지 URL을
  변경할 수 있습니다.
* burrito 네임스페이스에 로컬 yum 리포지터리 파드 배포합니다.
* happroxy에서 레지스트리 및 저장소 서비스 등록한다.
* GNSH(Install Graceful Node Shutdown Helper)를 설치합니다.

설치
^^^^^

landing 플레이북 실행.::

   $ ./run.sh landing

확인
^^^^^

제네시스 레지스트리 서비스가 컨트롤 노드에서 실행중인지 확인합니다.::

   $ sudo systemctl status genesis_registry.service gnsh.service
   genesis_registry.service - Geneis Registry service
   ...
    Active: active (running) since Fri 2023-09-22 14:39:41 KST; 3min 13s ago
   ...
   gnsh.service - Graceful Node Shutdown Helper
   ...
     Active: active (exited) since Fri 2023-09-22 14:42:36 KST; 19s ago

로컬 리포지토리 파드가 runnin 및 ready 인지 kube-system 네임스페이스에서
확인합니다..::

   $ sudo kubectl get pods -n kube-system
   NAME                        READY   STATUS    RESTARTS   AGE
   ...
   localrepo-c4bc5b89d-nbtq9   1/1     Running   0          3m38s

GNSH(Graceful Node Shutdown Helper) 서비스가 실행 중인지 확인합니다. .::

   $ sudo systemctl status gnsh.service
    gnsh.service - Graceful Node Shutdown Helper
      Loaded: loaded (/etc/systemd/system/gnsh.service; enabled; vendor preset: di>
      Active: active (exited) since Tue 2023-11-07 13:58:34 KST; 25min ago
     Process: 435851 ExecStart=/usr/bin/gnsh start (code=exited, status=0/SUCCESS)
    Main PID: 435851 (code=exited, status=0/SUCCESS)
       Tasks: 0 (limit: 100633)
      Memory: 0B
      CGroup: /system.slice/gnsh.service
   
   Nov 07 13:58:34 control1 systemd[1]: Starting Graceful Node Shutdown Helper...
   Nov 07 13:58:34 control1 gnsh[435851]: Uncordon my node control1.
   Nov 07 13:58:34 control1 gnsh[435853]: node/control1 already uncordoned
   Nov 07 13:58:34 control1 systemd[1]: Started Graceful Node Shutdown Helper.


축하합니다.! 

burrito 플랫폼 설치를 완료했습니다.

다음은 burrito 플랫폼에 OpenStack을 설치합니다.

Step.9 Burrito
+++++++++++++++++

burrito 설치 단계는 다음과 같은 작업을 수행합니다.

* rados gateway 사용자 (default: cloudpc)를 생성하고
  클라이언트 구성 (s3cfg).
* nova vnc TLS 인증서를 배포합니다.
* OpenStack 구성 요소를 배포합니다.
* nova ssh keypair를 생성하여 모든 컴퓨트 노드에 복사합니다.

설치
^^^^^

burrito 플레이북 실행.::

   $ ./run.sh burrito

확인
^^^^^

openstack 네임스페이스에서 모든 파드가 running, ready 상태인지 확인.::

   $ sudo kubectl get pods -n openstack
   NAME                                   READY   STATUS      RESTARTS   AGE
   barbican-api-664986fd5-jkp9x           1/1     Running     0          4m23s
   ...
   rabbitmq-rabbitmq-0                    1/1     Running     0          27m
   rabbitmq-rabbitmq-1                    1/1     Running     0          27m
   rabbitmq-rabbitmq-2                    1/1     Running     0          27m

축하합니다.!

burrito 쿠버네티스 플랫폼에서 OpenStack 설치를 완료했습니다.

Horizon
----------

horizon 대시보드는 컨트롤 노드에서 tcp 31000 수신합니다.

브라우저에서 horizon 대시보드에 연결하는 방법은 다음과 같습니다.

#. 브라우저를 엽니다.

#. keepalived_vip_svc 설정되어 있으면,
   https://<keepalived_vip_svc>:31000/ 연결합니다.

#. keepalived_vip_svc 설정되어 있지 않으면,
   https://<keepalived_vip>:31000/ 연결합니다.

#. 자체 서명된 TLS 인증서 수락 및 로그인.
   관리자 암호는 vault 실행할 때 설정 한 암호입니다.
   (오픈스택 관리자 비밀번호:).

다음은, btx(burrito toolbox)를 이용하여 기본적인 오픈스택 동작 테스트를 수행합니다.

BTX
---

BTX는 burrito 플랫폼을 위한 도구 상자입니다.
running 상태여야 합니다..::

   $ sudo kubectl -n openstack get pods -l application=btx
   NAME    READY   STATUS    RESTARTS   AGE
   btx-0   1/1     Running   0          36m

btx shell로 이동합니다. (bts).::

   $ . ~/.btx.env
   $ bts

openstack volume 서비스 상태를 확인합니다.::

   root@btx-0:/# openstack volume service list
   +------------------+------------------------------+------+---------+-------+----------------------------+
   | Binary           | Host                         | Zone | Status  | State | Updated At                 |
   +------------------+------------------------------+------+---------+-------+----------------------------+
   | cinder-scheduler | cinder-volume-worker         | nova | enabled | up    | 2023-05-31T12:05:02.000000 |
   | cinder-volume    | cinder-volume-worker@rbd1    | nova | enabled | up    | 2023-05-31T12:05:02.000000 |
   | cinder-volume    | cinder-volume-worker@netapp1 | nova | enabled | up    | 2023-05-31T12:05:07.000000 |
   +------------------+------------------------------+------+---------+-------+----------------------------+

다음은 hitachi storage backend의 볼륨 서비스 상태의 예입니다.::

   root@btx-0:/# o volume service list
   +------------------+------------------------------+------+---------+-------+----------------------------+
   | Binary           | Host                         | Zone | Status  | State | Updated At                 |
   +------------------+------------------------------+------+---------+-------+----------------------------+
   | cinder-scheduler | cinder-volume-worker         | nova | enabled | up    | 2023-12-12T07:46:59.000000 |
   | cinder-volume    | cinder-volume-worker@hitachi | nova | enabled | up    | 2023-12-12T07:46:56.000000 |
   +------------------+------------------------------+------+---------+-------+----------------------------+

* 모든 서비스가 활성화되어 있어야 합니다.
* ceph와 netapp storage backend를 모두 설정하면, 
  볼륨 서비스가 모두 활성화되고 output에서 up됩니다.
* cinder-volume-worker@rbd1는 ceph backend에 대한 서비스 입니다.
  그리고 cinder-volume-worker@netapp1 은 Netapp backend를 위한 서비스입니다.
* cinder-volumeworker@powerflex 는 Dell powerflex backend 서비스입니다.
* cinder-volumeworker@hitachi는 Hitachi backend를 위한 서비스입니다.

openstack 네트워크 에이전트 상태를 확인합니다.::

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
* overlay_iface_name을 null로 설정하면, 에이전트 타입에 'L3 agent'   
* is_ovs를 false로 설정하면, 에이전트 타입에 'Linux bridge agent'가 있어야합니다.  
* is_ovs를 true로 설정하면, 에이전트 타입에 'Open vSwitch agent'가 있어야 합니다.



openstack 컴퓨트 서비스 상태 확인.::

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

* 모든 서비스가 활성화 되어 있어야 합니다.
* 각 계산 노드에는 nova-compute 서비스가 있어야 합니다.

시험
++++

"btx --test" 명령

* provider 네트워크와 서브넷을 생성합니다.
  provider 네트워크를 생성할 때 주소 풀 범위를 묻습니다.
* cirros 이미지를 생성합니다.
* 보안 그룹 규칙을 추가합니다.
* flavor를 생성합니다.
* instance를 생성합니다.
* volume을 생성합니다.
* volume을 instance에 연결합니다.

모든 것이 정상적으로 진행되면, 다음과 같은 출력이 나타납니다.::

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

provider 네트워크 엑세스가 있는 머신에서 ssh를 사용하여 provider 네트워크
ip를 통해 인스턴스에 연결합니다.::

   (a node on provider network)$ ssh cirros@192.168.22.104
   cirros@192.168.22.104's password:
   $ ip address show dev eth0
   2: eth0:<BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1450 qdisc pfifo_fast qlen 1000
       link/ether fa:16:3e:ed:bc:7b brd ff:ff:ff:ff:ff:ff
       inet 192.168.22.104/24 brd 192.168.22.255 scope global eth0
          valid_lft forever preferred_lft forever
       inet6 fe80::f816:3eff:feed:bc7b/64 scope link
          valid_lft forever preferred_lft forever

비밀번호는 기본 cirros 비밀번호입니다.
(힌트: 비밀번호는 시카고 컵스 야구팀을 사랑하는 사람이 만든 것 같습니다.)




