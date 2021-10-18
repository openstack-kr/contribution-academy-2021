===================================================================
python-openstackclient, osc-sdk, openstacksdk 개발환경 만들기
===================================================================

*본 글에서는 오픈스택 프로젝트 개발 환경을 만들기 위한 방법을 정리하였습니다.
개발 환경을 만드는 예는 Pycharm 개발 도구를 사용하여 진행합니다.*

오픈스택은 Python 으로 구현된 IaaS 형태의 클라우드 컴퓨팅 오픈소스 프로젝트입니다.
핵심 구성요소와 API, SDK 등이 모두 Python 으로 개발되었으며, pep8 스타일을 따릅니다.

|

00. 개발환경
~~~~~~~~~~~~~~~~~~~~~~~

 - Python 3.7 이상
 - IDE: Pycharm

|

01. 프로젝트 Clone
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
openstackclient 는 명령줄 도구로 오픈스택 구성요소 CLI 를 관리합니다.
과거에는 nova / glance 와 같이 각각의 구성요소에 대한 이름을 가진 명령어로 API를 제어하였으나,
현재는 openstackclient 라는 통합 클라이언트 툴이 개발되어 openstack 이라는 명령어로 통합되었습니다.

|

**프로젝트 저장소**
 - **프로젝트 저장소**: https://opendev.org/
 - **프로젝트 미러 저장소**: https://github.com/openstack

|

개발환경을 만들기 위해서 다음의 3개 프로젝트를 클론 받아 진행합니다.

 - **python-openstackclient**: https://opendev.org/openstack/python-openstackclient
 - **osc-lib**: https://opendev.org/openstack/osc-lib
 - **openstacksdk**: https://opendev.org/openstack/openstacksdk

|

(참고) 아래의 문서는 참고해주세요.
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

**requirements.txt**

 - 프로젝트 개발에 있어 의존성 있는 라이브러리 버전 규정
 - openstack/requirements 저장소에서 공통 requirements 관리
 - test-requirements.txt 파일에 의존성 있는 라이브러리 정의

|

**tox.ini**

 - 오픈스택 프로젝트 저장소에서 tox.ini 에 실행할 내용을 정의하여 테스트 및 문서 빌드

|

02. Clone 받은 프로젝트 Attach
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
프로젝트 3개를 모두 클론 받았다면, Pycharm 의 하나의 프로젝트에 모두 Attach 합니다.

 .. code-block:: none

    git clone https://opendev.org/openstack/python-openstackclient
    git clone https://opendev.org/openstack/osc-lib
    git clone https://opendev.org/openstack/openstacksdk

|

 .. image:: ../images/project_setup_01.png

|

03. 파이썬 가상환경 virtualenv 생성
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
 - 프로젝트를 모두 Attach 했다면, Pycharm 에서 openstackcli 와 sdk 를 이용하기 위한 가상환경을 생성합니다.
 - 가상환경 내에서는 지역적으로 패키지를 관리할 수 있습니다. 이 외에도 가상환경에서 작업하는 이유는 다양합니다.
 - 윈도우 기준 File > Settings > python interpreter > 톱니바퀴 > Add 를 선택합니다.

 .. image:: ../images/project_setup_02.png

|

 - 설정은 python-openstackclient 프로젝트를 대상으로 추가합니다.
 - Base Interpreter 는 최소 Python 3.7 이상으로 선택합니다. 만약 설치되어 있지 않다면, `파이썬 설치 <https://www.python.org/>`_ 에서 최신 버전의 파이썬을 설치합니다.


 .. image:: ../images/project_setup_03.png

|

 .. image:: ../images/project_setup_04.png

|

 - 나머지 프로젝트(openstacksdk/osc-lib)의 인터프리터를 python-openstackclient 의 가상환경으로 선택합니다.
 - 파이참 상단의 Edit Configuration 또는 Add Configuration 에서 인터프리터를 선택할 수 있습니다.

 .. image:: ../images/project_setup_06.png

|

 .. image:: ../images/project_setup_07.png

|

 - python-openstackcli/venv 로 만든 가상환경을 activate 합니다.

 .. code-block:: none

    macOS/linux: source venv/bin/activate
    window: activate.bat

|

 - 각 프로젝트 폴더로 이동하여 기본 코드를 설치합니다.

 .. code-block:: none

    python setup.py develop

 .. code-block:: none

    (venv) C:openstack-team-blog\osc-lib>python setup.py develop
    (venv) C:openstack-team-blog\python-openstackclient>python setup.py develop
    (venv) C:openstack-team-blog\openstacksdk>python setup.py develop

|

 - 각 프로젝트마다 setup.py 설치가 완료되었다면, \'Finished processing dependencies for ~\' 라는 내용이 출력됩니다.

 .. image:: ../images/project_setup_08.png