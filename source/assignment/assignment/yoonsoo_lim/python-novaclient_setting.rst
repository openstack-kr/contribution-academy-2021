Python-novaclient 프로젝트 개발 환경 세팅하기
=====================================================================

1. git clone 받아야할 프로젝트 주소
---------------------------------------------
    * https://opendev.org/openstack/python-novaclient

    .. code-block:: bash

        * clone이 완료되면 branch를 wallay로 변경

        git checkout stable/wallaby

        $ git branche
        master
        * stable/wallaby


2. virtualenv 생성하기
--------------------------------

    * 전에 세팅 하던 방식처럼 Clone 받은 프로젝트를 pycharm에 attach 해주고 virtualenv를 생성해줍니다.

    .. image:: ./images/attach.png

----


    * 아래 사진에서 ``Add..`` 를 클릭해줍니다.

    .. image:: ./images/virtual-1.png

----


    * 아래에서 ``location`` 은 python-novaclient/venv로 설정해주고 ``Base interpreter`` 는 기존에 사용하시던 python3.7이상으로 선택해줍니다.

    .. image:: ./images/virtual-2.png

3. virtualenv에 python-novaclient 설치하기
--------------------------------------------

    * macOS

    .. code-block:: bash

        source venv/bin/activate

    * Windows

    .. code-block:: bash

        .\venv\Scripts\activate

----

    * 마지막으로 아래 명령어로 설치 시작!

    .. code-block:: bash

        python setup.py develop

4. cli 실행 환경 만들기
------------------------------

    * 아래에 Run 부분을 실행시켜주고 shell 환경변수를 수정해줍니다.

    .. image:: ./images/shell-run.png

----

    * 환경변수를 클릭하여 1번에 nova-shell입력(선택)

    .. image:: ./images/configuration.png

----

    * 2번은 아래에 기존에 있던 환경변수를 복사해와서 붙여넣기를 해줍니다.

    .. image:: ./images/configuration-2.png

----

    .. image:: ./images/configuration-3.png

----

    * 마지막으로 3번에 명령어를 입력하고 run을 하면 novaclient로 nova 명령어가 실행이 된다!.

    .. image:: ./images/nova-result.png

