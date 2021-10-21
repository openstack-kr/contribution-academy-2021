========================================
2week - CLI와 친해지기
========================================



1) 계정 인증
----------------
1.1 계정 인증 파일
''''''''''''''''''''
    * command로 사용자 생성, 볼륨 생성 등 하기 전엔 로그인 하듯이 최초의 한번만 실행 시킨다
    * /opt/stack/devstack으로 이동 후

    .. code-block:: bash

        $ source openrc admin admin
        $ env | grep OS
        OS_USER_DOMAIN_ID=default
        OS_AUTH_URL=http://211.37.148.135/identity
        OS_PROJECT_DOMAIN_ID=default
        OS_REGION_NAME=RegionOne
        OS_PROJECT_NAME=demo
        OS_IDENTITY_API_VERSION=3
        OS_TENANT_NAME=demo
        OS_AUTH_TYPE=password
        OS_PASSWORD=secret
        OS_USERNAME=admin
        OS_VOLUME_API_VERSION=3
        OS_CACERT=

    * 계정 인증을 안할 시 밑에처럼 오류가 뜬다.

    .. code-block:: bash

        $ openstack router create test_router
        Missing value auth-url required for auth plugin password


2) 인스턴스
----------------
2.1 인스턴스 구성 및 생성
'''''''''''''''''''''''''''''

    .. code-block:: bash

        $ openstack server create cirror_instance \
        > --flavor m1.tiny \
        > --image cirros-0.5.2-x86_64-disk \
        > --boot-from-volume 1 \
        > --network private

3) 이미지의 패스워드 변경
-------------------------
3.1 우분투 이미지 다운
''''''''''''''''''''''''

    .. code-block:: bash

        $ wget https://cloud-images.ubuntu.com/releases/bionic/release/ubuntu-18.04-server-cloudimg-amd64.img


3.2 virt-customize를 이용하여 이미지 편집
''''''''''''''''''''''''''''''''''''''''''''

    .. code-block:: bash

            $ sudo apt install libguestfs-tools


    .. code-block:: bash

            $ virt-customize -a ubuntu-18.04-server-cloudimg-amd64.img --root-password password:$PASSWD
            [ 0.0] Examining the guest ... [ 44.6] Setting a random seed
            virt-customize: warning: random seed could not be set for this type of
            guest
            [ 44.7] Setting the machine ID in /etc/machine-id
            [ 44.7] Setting passwords
            [ 71.6] Finishing off


3.3 virt-customize 오류해결 및 원인
''''''''''''''''''''''''''''''''''''''''
    * virt-customize는 가상 머신 수정할 때 사용하는 명령어이다.
    * 또한 이미지도 수정이 가능한데 이미지를 수정할 때 QEMU로 이미지의 커널을 부팅할 때 ‘vmlinuz’파일을 사용하기 때문에
      일반 사용자(현: stack)가 사용하게 되면 퍼미션 에러가 발생한다.

    .. code-block:: bash

        $ virt-customize -a ubuntu-18.04-server-cloudimg-amd64.img --root-password password:$PASSWD
        [ 0.0] Examining the guest ... virt-customize: error: libguestfs error: /usr/bin/supermin exited with
        error status 1.
        To see full error messages you may need to enable debugging.
        Do: export LIBGUESTFS_DEBUG=1 LIBGUESTFS_TRACE=1 ………
        If reporting bugs, run virt-customize with debugging enabled and include
        the complete output:
         virt-customize -v -x [...]

    * 오류의 발생을 해결하려면 아래처럼 vmlinuz에 읽기(r)권한을 주거나 sudo 사용 또는 root사용자로 실행하여야 한다.

    .. code-block:: bash

        $ sudo chmod 0644 /boot/vmlinuz*


4) 인스턴스 생성
-------------------------
4.1 이미지 등록
''''''''''''''''''''''''

    .. code-block:: bash

        $ openstack image create ubuntu-18.04-server-cloudimg-amd64.img \
        > --file ubuntu-18.04-server-cloudimg-amd64.img \
        > --disk-format qcow2


    .. code-block:: bash

        $ openstack image list
        +--------------------------------------+----------------------------------------+-----------------------------+
        | ID                                   | Name                                   | Status                      |
        +--------------------------------------+----------------------------------------+-----------------------------+
        | 67b7ef06-b427-4ce1-94e8-dba…3474560  | cirros-0.5.2-x86_64-disk               | active                      |
        | f9da9de3-401b-42ad-be5b-8d…999d5df   | ubuntu-18.04-server-cloudimg-amd64.img | active                      |
        +--------------------------------------+----------------------------------------+-----------------------------+


4.2 인스턴스 생성 및 noVNC 접속
'''''''''''''''''''''''''''''''

    .. code-block:: bash

        $ openstack server create ubuntu-test \
        > --flavor m1.small \
        > --image f9da9de3-401b-42ad-be5b-8d538999d5df \
        > --network private \
        > --boot-from-volume 10

    .. code-block:: bash

        $ openstack server list
        +-----------------+-------------+---------+---------------------------+--------------------------+----------+
        | ID              | Name        | Status | Networks                   | Image                    | Flavor   |
        +---------------- +-------------+--------+--------------------------------+----------------------+----------+
        | bab18…58818     | ubuntu-test | ACTIVE | private=10.0.0.12, fd7…fbb | N/A (booted from volume) | m1.small |
        +-----------------+-------------+--------+----------------------------+--------------------------+----------+

    .. code-block:: bash

        $ openstack console url show ubuntu-test
        +----------+---------------------------------------------------------------------+
        | Field    | Value                                                               |
        +----------+---------------------------------------------------------------------+
        | protocol | vnc                                                                 |
        | type     | novnc                                                               |
        | url      | http://211.37.148.135:6080/vnc_…931f153-6de9-46f1-a49a-dc34001cb2e5 |
        +----------+---------------------------------------------------------------------+



5) Floating IP 생성 및 인스턴스에 할당
--------------------------------------
    * 인스턴스를 생성 할때는 내부ip만 할당을 해 주었고 외부와 통신을 할라면 외부IP(Floating IP)를 할당 해야한다.

    .. code-block:: bash

        $ openstack floating ip create public
        +-----------------------+---------------------------------------+
        | Field                 | Value                                 |
        +-----------------------+---------------------------------------+
        | created_at            | 2021-08-12T06:50:19Z                  |
        …
        | floating_ip_address   | 192.168.100.145                       |
        | floating_network_id   | bfda2e10-fc47-4c8f-90a4-06c43a364b65  |
        | id                    | d41adb91-20ab-4ecc-9a10-5576bdd07663  |
        | name                  | 192.168.100.145                       |
        …
        | updated_at            | 2021-08-12T06:50:19Z                  |
        +-----------------------+---------------------------------------+

        .. code-block:: bash

            $ openstack floating ip list
            +---------+-----------------------+------------------+------+------------------+----------------+
            | ID      | Floating IP Address   | Fixed IP Address | Port | Floating Network | Project        |
            +---------+-----------------------+------------------+------+------------------+----------------+
            | d4…663  | 192.168.100.145       | None             | None | bfda2…43a364b65 | 69973fe…04b574  |
            +---------+-----------------------+------------------+------+------------------+----------------+

        * 방금 생성된 Floating IP(형관펜 부분)를 이제 인스턴스에 연결을 해본다.

        .. code-block:: bash

            $ openstack server add floating ip ubuntu-test 192.168.100.145

        .. code-block:: bash

            $ openstack server show ubuntu-test
            +------------+--------------------------------------------------------------------------------+
            | Field      | Value                                                                          |
            +------------+--------------------------------------------------------------------------------+
            ……
            | addresses  | private=10.0.0.12, 192.168.100.145, fd76:952b:1ec0:0:f816:3eff:fee5:fbbb       |
            ……
            +------------+--------------------------------------------------------------------------------+



6) private 네트워크 생성 후 public network와 라우터로 연결
--------------------------------------------------------------
6.1 private 네트워크 생성
''''''''''''''''''''''''''''

    .. code-block:: bash

        $ openstack network create private-01

    .. code-block:: bash

        $ openstack subnet create private-01-sub \
        > --network private-01 \
        > --subnet-range 10.8.0.0/24 \
        > --gateway 10.8.0.1

6.2 Router 생성 및 확인
'''''''''''''''''''''''

    * 생성한 프로젝트 네트워크(private)와 외부 네트워크(public)을 연결해주는 라우터가 필요하다.

    .. code-block:: bash

        $ openstack router create router-01

6.3 외부 게이트웨이 추가
''''''''''''''''''''''''''''

    * 생성한 라우터에 외부 게이트웨이를 추가한다.

    .. code-block:: bash

        $ openstack router set router-01 --external-gateway public



6.4 내부 게이트웨이 추가
'''''''''''''''''''''''''''

    * 생성한 라우터에 내부 인터페이스스를 추한다.

    .. code-block:: bash

        $ openstack router add subnet router-01 private-01-sub

6.5 라우터 확인
'''''''''''''''''

    .. code-block:: bash

        $ openstack router show router-01
        +---------------------------+----------------------------------------------------------------------------+
        | Field                     | Value                                                                      |
        +---------------------------+----------------------------------------------------------------------------+
        | admin_state_up            | UP                                                                         |
        | availability_zone_hints   |                                                                            |
        | availability_zones        | nova                                                                       |
        | created_at                | 2021-08-12T07:54:06Z                                                       |
        | description               |                                                                            |
        | distributed               | False                                                                      |
        | external_gateway_info     | null                                                                       |
        | flavor_id                 | None                                                                       |
        | ha                        | False                                                                      |
        | id                        | 34a54d42-4faa-4667-9431-08e5f12e7b52                                       |
        | interfaces_info           | [{"port_id": "1f…a4", "ip_address": "10.8.0.1", "subnet_id": "5e…98"}]     |
        | name                      | router-01                                                                  |
        | project_id                | 69973fe10b1540f0a5810883b604b574                                           |
        | revision_number           | 2                                                                          |
        | routes                    |                                                                            |
        | status                    | ACTIVE                                                                     |
        | tags                      |                                                                            |
        | updated_at                | 2021-08-12T07:54:34Z                                                       |
        +---------------------------+----------------------------------------------------------------------------+

