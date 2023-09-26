오류 수정사항
==================

placement와 barbican replica 이슈
-------------------------------------

* 영향 받는 버전: 1.2.3 및 이전 버전

OpenStack placement와 barbican API 파드의 replicas는 1로 설정되어 있습니다.
만약 여러 개의 control 노드가 있다면 최소 2로 설정해야 합니다.

아직 Burrito가 설치되지 않은 경우,
:download:`the patch <../_static/00-patch-placement-barbican-replicas.txt>` 다운로드 링크를 클릭하여 <burrito_source_dir>/patches/ 디렉토리에 넣습니다.
그리고 설치 절차를 따르십시오.
(prepare.sh 스크립트를 실행하면 패치가 될 것입니다.)

이미 Burrito가 설치된 경우, btx 쉘로 이동하여
placement와 barbican replica를 설정하세요.


::

    root@btx-0:/# k scale --replicas=2 deploy barbican-api 
    root@btx-0:/# k scale --replicas=2 deploy placement-api

