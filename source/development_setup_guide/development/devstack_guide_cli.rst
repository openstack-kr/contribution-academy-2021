==========================================================
DevStack 에서 CLI 이용하기 및 openrc 스크립트 내용 분석
==========================================================

openrc
~~~~~~~~~~

 - devstack 은 환경변수를 계정에 맞게 설정해주는 openrc 라는 스크립트를 제공합니다.
 - openrc 는 OpenStack 을 CLI 로 사용하기 위한 인증을 구성하는 스크립트입니다.
 - 아래와 같이 입력하면 admin 계정으로 환경변수가 설정되어 CLI 를 이용할 수 있습니다. 다만, 현재 세션에서만 적용되는 로컬 환경변수로 설정 정보를 임시적으로 설정하기에 현재 세션이 아닌 다른 세션으로 연결했을 경우 다시 환경변수를 잡아야 합니다.
 - openrc 파일을 source 명령어로 파일을 로드하여 시스템 리부팅 없이 즉시 적용합니다.

 .. code-block:: none

    source openrc [username] [project name]

    example)
    $ source openrc admin admin

|

**환경변수 적용 확인**
 - openrc 를 이용해 설정한 환경변수는 export 명령어로 현재 시스템에 선언된 환경변수 목록을 확인할 수 있습니다.

 .. code-block:: none

    $ export

|

openrc 스크립트 분석
~~~~~~~~~~~~~~~~~~~~~~~~~

 - openrc 는 username, project 두 개의 인자를 받습니다.

 .. code-block:: none

    source openrc [$username] [$projectname]
    => source openrc [$1] [$2]

|

 - openrc 스크립트를 로드하면서 전달된 두 개의 인자는 스크립트 **내 $OS_USERNAME**, **$OS_PROJECT_NAME** 변수에 담깁니다.

 .. code-block:: none

    if [[ -n "$1" ]]; then
        OS_USERNAME=$1
    fi
    if [[ -n "$2" ]]; then
        OS_PROJECT_NAME=$2
    fi

|

 - 두 변수에 값을 담고, 스크립트 파일의 절대 경로를 얻습니다. source 명령어로 스크립트를 로드할 때 **$BASH_SOURCE** 변수를 이용해 실행 파일이 있는 Path 를 구할 수 있습니다.
 - 현재 경로를 DC_DIR 변수에 담고 이 경로 정보로 functions, stackrc, .stackenv, tls 스크립트 파일을 로드합니다.

 .. code-block:: none

    # Find the other rc files
    RC_DIR=$(cd $(dirname "${BASH_SOURCE:-$0}") && pwd)

    # Import common functions
    source $RC_DIR/functions

    # Load local configuration
    source $RC_DIR/stackrc

    # Load the last env variables if available
    if [[ -r $RC_DIR/.stackenv ]]; then
        source $RC_DIR/.stackenv
        export OS_CACERT
    fi

    # Get some necessary configuration
    source $RC_DIR/lib/tls

|

 - 위와 같이 절대 경로를 얻는 과정은 다음과 같은 스크립트를 작성하여 테스트 할 수 있습니다.

 .. code-block:: none

    #!/bin/sh

    RC_DIR=$(cd $(dirname "${BASH_SOURCE}") && pwd)
    echo 'DIR: ' $RC_DIR

|

 - KeyStone 인증을 위한 환경변수를 설정합니다.
 - KeyStone 은 토큰 및 서비스에 대한 사용자/테넌트가 접근할 수 있는 모든 서비스에 대한 인증을 하고 토큰을 발행합니다.
 - **OS_PROJECT_NAME(구 OS_TENANT_NAME)**: 키스톤은 현재 project 라는 용어로 표준화가 되었습니다. 따라서 이전 버전과의 호환성을 위해 이전 용어인 tenant 는 계속 사용됩니다.
 - **OS_USERNAME**: CLI 를 실행하기 위한 사용자입니다.
 - **OS_PASSWORD**: Keystone 인증을 위한 환경변수에 패스워드입니다. 이 패스워드는 local.conf 설정을 따릅니다. (이 패스워드는 평문으로 저장되며, 암호화되지 않습니다.)
 - **OS_REGION_NAME**: 클라우드 인프라가 위치한 국가 혹은 지역에 대한 별칭입니다. 기본값은 RegionOne 입니다.

 .. code-block:: none

    export OS_PROJECT_NAME=${OS_PROJECT_NAME:-demo}

    echo "WARNING: setting legacy OS_TENANT_NAME to support cli tools."
    export OS_TENANT_NAME=$OS_PROJECT_NAME

    # In addition to the owning entity (project), nova stores the entity performing
    # the action as the **user**.
    export OS_USERNAME=${OS_USERNAME:-demo}

    # With Keystone you pass the keystone password instead of an api key.
    # Recent versions of novaclient use OS_PASSWORD instead of NOVA_API_KEYs
    # or NOVA_PASSWORD.
    export OS_PASSWORD=${ADMIN_PASSWORD:-secret}

    # Region
    export OS_REGION_NAME=${REGION_NAME:-RegionOne}

|

 - **$HOST_IP** 를 설정하여 API 엔드포인트 호스트를 지정합니다.
 - **$OS_IDENTITY_API_VERSION** 변수에는 Keystone 의 버전 정보가 담깁니다.

 .. code-block:: none

    if [[ $SERVICE_IP_VERSION == 6 ]]; then
        HOST_IPV6=${HOST_IPV6:-::1}
        SERVICE_HOST=${SERVICE_HOST:-[$HOST_IPV6]}
        GLANCE_HOST=${GLANCE_HOST:-[$HOST_IPV6]}
    else
        HOST_IP=${HOST_IP:-127.0.0.1}
        SERVICE_HOST=${SERVICE_HOST:-$HOST_IP}
        GLANCE_HOST=${GLANCE_HOST:-$HOST_IP}
    fi

    # Identity API version
    export OS_IDENTITY_API_VERSION=${IDENTITY_API_VERSION:-3}

    # Ask keystoneauth1 to use keystone
    export OS_AUTH_TYPE=password

 |

 - 인증 URL과 관련된 **$OS_AUTH_URL** 변수에는 사용자 인증을 위한 Identity API 엔드포인트 URL이 설정됩니다.
 - (참고: Keystone 은 사용자 인증을 위해 Identity API 엔드포인트 URL에 POST 방식으로 인증 정보를 전달합니다. 이 API에서 openstackclient 를 사용하기 위해서는 사용자 및 프로젝트에 대한 도메인 정보가 필요하고, API를 요청하는 사용자와 컴포넌트들은 키스톤에 대한 API 엔드포인트 정보를 알고 있어야 합니다.)
 - **$OS_CAERT** 환경변수는 SSL/TLS(보안 인증서) 적용을 위한 환경변수입니다.

 .. code-block:: none

    # If you don't have a working .stackenv, this is the backup position
    KEYSTONE_BACKUP=$SERVICE_PROTOCOL://$SERVICE_HOST:5000
    KEYSTONE_SERVICE_URI=${KEYSTONE_SERVICE_URI:-$KEYSTONE_BACKUP}

    export OS_AUTH_URL=${OS_AUTH_URL:-$KEYSTONE_SERVICE_URI}

    # Currently, in order to use openstackclient with Identity API v3,
    # we need to set the domain which the user and project belong to.
    if [ "$OS_IDENTITY_API_VERSION" = "3" ]; then
        export OS_USER_DOMAIN_ID=${OS_USER_DOMAIN_ID:-"default"}
        export OS_PROJECT_DOMAIN_ID=${OS_PROJECT_DOMAIN_ID:-"default"}
    fi

    # Set OS_CACERT to a default CA certificate chain if it exists.
    if [[ ! -v OS_CACERT ]] ; then
        DEFAULT_OS_CACERT=$INT_CA_DIR/ca-chain.pem
        # If the file does not exist, this may confuse preflight sanity checks
        if [ -e $DEFAULT_OS_CACERT ] ; then
            export OS_CACERT=$DEFAULT_OS_CACERT
        fi
    fi

    # Currently cinderclient needs you to specify the *volume api* version. This
    # needs to match the config of your catalog returned by Keystone.
    export CINDER_VERSION=${CINDER_VERSION:-3}
    export OS_VOLUME_API_VERSION=${OS_VOLUME_API_VERSION:-$CINDER_VERSION}

|
