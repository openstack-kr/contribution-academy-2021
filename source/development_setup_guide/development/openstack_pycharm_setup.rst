==========================================================
PyCharm 으로 openstackcli 실행 환경 만들기
==========================================================

*본 글에서는 CLI 를 PyCharm 에서 실행하기 위한 방법을 작성하였습니다. 개발환경 구성 및 CLI 에 대한 내용은 이전 문서를 참고해주세요.*

openstack 명령어의 main은 python-openstackclient/openstackclient/shell.py 입니다.
PyCharm 에서 shell.py 파일을 임의로 실행시켜 CLI 실행 환경을 만듭니다.

|

PyCharm 에서 parameters 와 Environment variables 설정하기
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
 - python-openstackclient/openstackclient 내의 shell.py 파일을 우측 클릭하여, Run \'shell\' 버튼을 선택해주세요.

 .. image:: ../images/project_cli_01.png

|

 - 해당 파일이 실행이 되었다면, 오른쪽 상단의 shell 실행 환경을 수정하기 위해서 \'Edit Configration\' 를 선택합니다.

 .. image:: ../images/project_cli_02.png

|

 - 여기서 수정할 부분은 parameters 와 Environment variables 를 수정합니다.
 - parameters 에는 openstack 명령어의 인자값을 넣어줍니다. (예: server list / image list)

 .. image:: ../images/project_cli_03.png

|

 .. image:: ../images/project_cli_04.png

|

 - environment variable 에는 devstack의 openrc 스크립트 파일로 설정했던 동일한 환경 변수를 설정합니다.
 - 환경변수와 관련된 상세한 내용은 팀 블로그의 **\'개발환경 만들기\'** > **\'DevStack 에서 CLI 이용하기 및 openrc 내용 분석\'** 문서를 참고해주세요.
 - 설정해야 하는 값은 다음과 같습니다.

 .. code-block:: none

    [키 / 값]
    OS_PROJECT_NAME / admin
    OS_TENANT_NAME / admin
    OS_USERNAME / admin
    OS_PASSWORD / secret
    OS_REGION_NAME / RegionOne
    OS_REGION_NAME / 3
    OS_AUTH_TYPE / password
    OS_AUTH_URL / http://IP/identity
    OS_USER_DOMAIN_ID / default
    OS_PROJECT_DOMAIN_ID / default
    OS_VOLUME_API_VERSION / 3

|

 - 이 설정은 리눅스 서버에서 openrc 스크립트를 로드하고 export 명령으로 확인이 가능합니다.

 .. code-block:: none

    source openrc admin admin
    export | grep "OS"

 .. code-block:: none

    example)

    stack@doa-wallaby-2:~/devstack$ source openrc admin admin

    stack@doa-wallaby-2:~/devstack$ export | grep "OS"
    declare -x OS_AUTH_TYPE="password"
    declare -x OS_AUTH_URL="http://컨트롤러노드IP/identity"
    declare -x OS_CACERT=""
    declare -x OS_IDENTITY_API_VERSION="3"
    declare -x OS_PASSWORD="secret"
    declare -x OS_PROJECT_DOMAIN_ID="default"
    declare -x OS_PROJECT_NAME="admin"
    declare -x OS_REGION_NAME="RegionOne"
    declare -x OS_TENANT_NAME="admin"
    declare -x OS_USERNAME="admin"
    declare -x OS_USER_DOMAIN_ID="default"
    declare -x OS_VOLUME_API_VERSION="3"

|

실행하기
~~~~~~~~~~~~~~~~~~~~~~

 - parameters 와 Environment variables 를 모두 설정한 다음 **Run** 을 눌러 정상적으로 의도한 명령어가 실행되는지 확인합니다.
 - 만일, 아래와 같이 실행이 되지 않는다면, 패스워드 혹은 컨트롤러 노드에 대한 아이피를 정확히 입력하였는지 환경변수에 대한 값들을 검토합니다.
 - 정상적으로 실행된다면, 다음과 같이 PyCharm 하단에서 명령어의 실행 결과가 출력됩니다.

 .. image:: ../images/project_cli_05.png
