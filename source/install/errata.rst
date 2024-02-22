Errata
=======

gnsh.service 시작될 때 Uncordon 문제
----------------------------------------

* 영향 받는 버전: aster 전체 버전

Gnsh는 Graceful Node Shutdown Helper 프로그램 입니다.
노드가 정상적으로 종료될 때 노드를 cordons 및 drains하고,
노드가 부팅될 때 노드를 uncordons 합니다.

부팅 시 gnsh systemd 서비스가 시작될 때,
/usr/bin/gnsh 스크립트는 노드를 한번만 uncordon 하려고 시도하며,
만약 kube-apiserver가 아직 준비되지 않았다면 실패할 수 있습니다.

노드가 uncordon 되지 않은 경우, 30초 동안 10번 시도하여 노드를
uncordon 하도록 /usr/bin/gnsh 스크립트를 수정했습니다.

다운로드 :download:`수정된 gnsh script <../_static/gnsh>` 를 다운로드하여
모든 쿠버네티스 노드 /usr/bin/ 디렉토리에 넣으시면 됩니다.::

    $ chmod +x gnsh
    $ sudo cp gnsh /usr/bin/

그러면 다음 부팅 시에 적용될 것입니다.