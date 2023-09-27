Storageclass Backend 추가
=============================



burrito에서 현재 사용 설치 가능한 Storageclass Backend는 ceph, netapp, powerflex가 있습니다.


Storageclass Backend 특징
------------------------------

Storageclass Backend에 ceph, netapp, powerflex는 각각의 특징이 있습니다.

**Ceph**
   
   분산 스토리지 시스템: Ceph는 분산 스토리지 시스템으로, 대규모 클러스터를 통해 데이터를 저장하고 관리합니다.
   
   강력한 확장성: Ceph는 노드를 추가함으로써 스토리지 용량과 성능을 확장할 수 있습니다.
   
   객체, 블록 및 파일 스토리지: Ceph는 객체 스토리지, 블록 스토리지 및 파일 스토리지를 모두 지원하므로 다양한 스토리지 요구 사항을 충족시킬 수 있습니다.

**NetApp**
   
   전통적인 스토리지 벤더: NetApp은 전통적인 스토리지 제공 업체로서 기업용 스토리지 솔루션을 제공합니다.
   
   고성능 및 데이터 관리 기능: NetApp 스토리지는 뛰어난 성능과 데이터 관리 기능을 제공하여 대규모 데이터 센터 환경에서 많이 사용됩니다.
   
   스냅샷, 복제 및 데이터 복구: NetApp은 데이터의 스냅샷, 복제 및 복구를 관리하기 위한 기능을 제공하여 데이터 보호와 관리를 강화합니다.

**PowerFlex**
   
   하이퍼 컨버지드 인프라 (HCI): PowerFlex는 하이퍼 컨버지드 인프라 솔루션으로, 컴퓨팅, 스토리지 및 네트워킹을 통합한 솔루션을 제공합니다.
   
   소프트웨어 정의 스토리지: PowerFlex는 소프트웨어로 정의된 스토리지 아키텍처를 기반으로 하며, 가상화 및 컨테이너 환경에서 유연한 스토리지 관리를 제공합니다.
   
   스케일 아웃 및 자동화: PowerFlex는 스토리지 자원의 스케일 아웃과 자동화를 지원하여 요구 사항에 따라 리소스를 증가시킬 수 있습니다.




만약 netapp을 추가 하고 싶다면
---------------------------------

vars yml파일을 수정합니다.

::

   ### storage
   # storage backends: ceph and(or) netapp
   # If there are multiple backends, the first one is the default backend.
   storage_backends:
   - 기존 storage
   - netapp(추가)


group_vars/all/netapp_vars.yml을 수정합니다.

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



netapp playbook 실행합니다.

:: 

   $ ./run.sh netapp

   PLAY RECAP *************************************************************************
   compute1                   : ok=2    changed=0    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0
   compute2                   : ok=2    changed=0    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0
   control1                   : ok=12   changed=10   unreachable=0    failed=0    skipped=0    rescued=0    ignored=0
   control2                   : ok=2    changed=0    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0
   control3                   : ok=2    changed=0    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0

   ===============================================================================
   burrito.netapp : netapp | install trident ---------------------------------- 94.99s
   burrito.netapp : netapp | download and unarchive the trident package -------- 7.99s
   burrito.netapp : netapp | copy manifest files ------------------------------- 6.97s
   Gathering Facts ------------------------------------------------------------- 6.75s
   burrito.netapp : netapp | Install the NFS tools ----------------------------- 5.27s
   burrito.netapp : netapp | templating config files --------------------------- 2.43s
   burrito.netapp : netapp | create trident namespace -------------------------- 1.42s
   burrito.netapp : netapp | create a backend ---------------------------------- 1.40s
   burrito.netapp : netapp | copy tridentctl binary to bin_dir ----------------- 1.27s
   burrito.netapp : netapp | create a storageclass ----------------------------- 1.25s
   burrito.netapp : netapp | check if trident is already installed ------------- 0.93s
   burrito.netapp : netapp | create netapp artifacts directory ----------------- 0.63s


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


cinder chart에도 netapp을 배포해야 합니다.


::


   $ ./scripts/burrito.sh install cinder

   PLAY [Deploy ssh key pair in compute nodes.] ***************************************

   PLAY RECAP *************************************************************************
   control1                   : ok=12   changed=4    unreachable=0    failed=0    skipped=4    rescued=0    ignored=0
   control2                   : ok=1    changed=0    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0
   control3                   : ok=1    changed=0    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0

   ===============================================================================
   burrito.openstack : OpenStack | deploy osh charts ------------------------- 188.71s
   Gathering Facts ------------------------------------------------------------- 6.97s
   burrito.openstack : OpenStack | make sure to set up control plane label ----- 1.79s
   burrito.openstack : OpenStack | create openstack namespace ------------------ 1.46s
   burrito.openstack : OpenStack | templating osh values ----------------------- 1.08s
   burrito.openstack : OpenStack | make sure to set up compute node label ------ 1.00s
   burrito.openstack : OpenStack | create openstack artifacts directory -------- 0.57s
   burrito.openstack : OpenStack | symlink helm-toolkit dependency chart for osh --- 0.43s
   burrito.openstack : OpenStack | create dependency chart directory for osh --- 0.38s
   burrito.openstack : OpenStack | add ceph-provisioners to osh_infra_charts if ceph in storage_backends --- 0.13s
   burrito.openstack : OpenStack | add openvswitch to osh_infra_charts if neutron_ml2_plugin is ovs --- 0.11s
   burrito.openstack : OpenStack | debug --------------------------------------- 0.08s
   burrito.openstack : OpenStack | get cinder keyring -------------------------- 0.07s
   burrito.openstack : OpenStack | get admin keyring --------------------------- 0.06s
   burrito.openstack : OpenStack | set fact for admin keyring ------------------ 0.06s
   burrito.openstack : OpenStack | set fact for cinder keyring ----------------- 0.06s



openstack 볼륨 서비스 상태를 확인해야 합니다.

::

   root@btx-0:/# openstack volume service list
   +------------------+--------------------------------+------+---------+-------+----------------------------+
   | Binary           | Host                           | Zone | Status  | State | Updated At                 |
   +------------------+--------------------------------+------+---------+-------+----------------------------+
   | cinder-scheduler | cinder-volume-worker           | nova | enabled | up    | 2023-09-27T03:09:49.000000 |
   | cinder-scheduler | cinder-volume-worker           | nova | enabled | down  | 2023-09-27T03:06:33.000000 |
   | cinder-volume    | cinder-volume-worker@netapp1   | nova | enabled | up    | 2023-09-27T03:09:44.000000 |
   +------------------+--------------------------------+------+---------+-------+----------------------------+



만약 powerflex를 추가 하고 싶다면
-------------------------------------