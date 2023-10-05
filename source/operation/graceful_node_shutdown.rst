정상적인 노드 종료
========================

Kubernetes는 1.21 버전부터 Graceful Node Shutdown 기능을 갖고 있습니다. 
(https://kubernetes.io/docs/concepts/architecture/nodes/#graceful-node-shutdown)

이 기능은 노드 종료 이벤트를 감지할 때 systemd inhibitor lock을 사용하여 종료 프로세스를 지연시킵니다.

현재 burrito의 Kubernetes 버전은 1.24.14이며, 자연스러운 노드 종료 기능은 기본적으로 활성화되어 있습니다.

그러나 이 기능은 테스트를 진행한 결과 안정적이지 않습니다. 이 기능과 관련된 많은 이슈가 있었습니다.

다음은 그 중 몇 가지입니다.

* https://github.com/kubernetes/kubernetes/issues/112443
* https://github.com/kubernetes/kubernetes/issues/110755
* https://github.com/kubernetes/kubernetes/issues/107158

그래서 우리는 Graceful Node Shutdown Helper (GNSH, pronounce '지엔쉬')를 개발하고 burrito 1.2.4에 burrito.gnsh 역할을 추가했습니다.

GNSH는 노드가 시작되거나 종료/재부팅될 때 실행되는 짧은 스크립트입니다.

노드가 종료되거나 재부팅될 때, 다음은 노드에서 파드를 밀어내는 프로세스입니다.

#. kubelet은 부팅 시 종료를 300초 지연시키기 위해 inhibitor lock을 등록합니다.
#. systemctl [poweroff|reboot] 또는 전원 버튼을 누릅니다, etc...
#. kubelet은 종료 이벤트를 감지합니다.
#. kubelet Shutdown Manager는 노드 상태를 NotReady로 변경합니다.
#. kubelet Shutdown Manager는 파드를 종료하고 파드 상태를 API 서버에 업데이트합니다.
#. Kubelet Shutdown Manager는 종료 이벤트 처리를 완료하고 delay inhibitor lock을 해제합니다.
#. 이제 systemd 데몬에 의해 GNSH 중지 프로세스가 진행됩니다.
#. GNSH는 노드의 drain을 처리하기 위한 프로세스를 실행합니다.
#. 노드가 격리되어 노드 상태가 SchedulingDisabled로 변경됩니다.
#. 노드에 남아 있는 경우 정적(static) 및 데몬셋(daemonset)을 제외한 모든 Pod가 제거됩니다.
#. GNSH는 제거 프로세스를 완료합니다.
#. 그리고 systemd가 나머지 종료 절차를 수행합니다.

노드를 전원을 끄는 경우 로그는 다음과 같습니다.

::

    Sep  9 17:45:37 control3 systemd-logind[666]: Power key pressed.
    Sep  9 17:45:37 control3 systemd-logind[666]: Powering Off...
    Sep  9 17:45:37 control3 kubelet[973]: I0909 17:45:37.860271     973 nodeshutdown_manager_linux.go:262] "Shutdown manager detected new shutdown event, isNodeShuttingDownNow" event=true
    ...
    Sep  9 17:46:08 control3 kubelet[973]: I0909 17:46:08.890638     973 nodeshutdown_manager_linux.go:324] "Shutdown manager completed processing shutdown event, node will shutdown shortly"
    Sep  9 17:46:08 control3 systemd-logind[667]: System is powering down.
    ...
             Stopping Graceful Node Shutdown Helper...
    [  920.090119] gnsh[13450]: Show the node status.
    [  920.209689] gnsh[13452]: NAME       STATUS     ROLES           AGE    VERSION
    [  920.211873] gnsh[13452]: control3   NotReady   control-plane   3d5h   v1.24.14
    [  925.214986] gnsh[13450]: Let's drain my node control3
    [  925.338511] gnsh[13483]: node/control3 cordoned
    [  925.376366] gnsh[13483]: evicting pod kube-system/nodelocaldns-rxnzq
    [  925.378692] gnsh[13483]: evicting pod kube-system/kube-proxy-fs9f7
    [  925.381179] gnsh[13483]: evicting pod kube-system/calico-node-sr9qk
    ...
    [  925.421417] gnsh[13483]: pod/calico-node-sr9qk evicted
    [  925.424240] gnsh[13483]: pod/kube-proxy-fs9f7 evicted
    [  925.427133] gnsh[13483]: pod/nodelocaldns-rxnzq evicted
    [  925.429745] gnsh[13483]: node/control3 drained
    [  930.450105] gnsh[13450]: Graceful Node Shutdown Helper (GNSH) completed a drain process
    [  OK  ] Stopped Graceful Node Shutdown Helper.
    ...
             Stopping Kubernetes Kubelet Server...
    ...
    [  OK  ] Stopped Kubernetes Kubelet Server.
    ...
             Stopping containerd container runtime...
    ...
    [  OK  ] Stopped containerd container runtime.
    ...
    [  933.761678] reboot: Power down


노드가 시작되면 kubelet은 노드를 Ready 상태로 만들고, GNSH는 노드를 스케줄 가능하게 만듭니다.


경고(Caveat)
--------------

systemctl 명령에 의해서만 kubelet 노드 종료 관리자가 트리거됩니다.

* systemctl poweroff
* systemctl reboot
* systemctl halt

다음 명령어는 kubelet 노드 종료 관리자를 트리거하지 않습니다. 

* shutdown -h now
* shutdown -P now
* poweroff
* reboot

Rocky Linux의 systemd 버전(v239)은 레거시 명령어에 대한 dbus 신호를 방출하지 않으므로 kubelet의 inhibitor가 존중되지 않습니다.

따라서 노드를 종료할 때 systemctl 명령어를 사용하십시오.
