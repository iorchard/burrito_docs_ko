오류 수정사항
================

체크리스트 이슈
--------------------

* 영향 받는 버전: 1.2.2

Burrito를 설치하기 전에 미리 확인하는 checklist.yml playbook에는 storage 노드를 포함하여 모든 인터페이스가 작동하는지 확인하는 작업이 있습니다.
storage 노드에 overlay 또는 provider 인터페이스가 없을 수 있으므로 storage 노드는 이 작업에서 건너뛰어야 합니다.

:download:`patch <../_static/00-patch-checklist.txt>`  다운로드 링크를 클릭하여 <burrito_source_dir>/patches/ 디렉토리에 넣습니다.


패치를 적용합니다.

::

    $ cd <burrito_src_dir>
    $ ./scripts/patch.sh 