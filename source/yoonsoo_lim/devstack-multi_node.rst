[devstack] Multi Node Install Guide
=====================================================================

1. 프로젝트 개요
------------------------------
    * ``vmware`` 를 기준으로 실습하였습니다.
    * ``Multi Node`` 구성은 아래 링크를 참고하여 구축하였습니다.
    * https://docs.openstack.org/devstack/latest/guides/neutron.html

    * 어떤 Node에서 명령어를 실행해야 하는지 각각 표기 해놨습니다.

--------

1.1 설치환경 구성표
''''''''''''''''''''''''''



    * ``NIC 1`` 은 ssh 접속을 위한 IP이고 ``NIC 2`` 는 오픈스택 구축에 사용할 IP입니다.
    * ``RAM`` 사양은 8기가정도면 적당한거 같습니다.

    +-------+-----------------+-----------------+
    |       | Controller Node | Compute Node    |
    +=======+=================+=================+
    | ens33 | 192.168.184.100 | 192.168.184.101 |
    +-------+-----------------+-----------------+
    | ens34 |   172.16.0.11   |   172.16.0.12   |
    +-------+-----------------+-----------------+
    |  CPU  |      2 core     |      2 core     |
    +-------+-----------------+-----------------+
    |  RAM  |       12 GB     |       12 GB     |
    +-------+-----------------+-----------------+
    |  HDD  |      100 GB     |       50 GB     |
    +-------+-----------------+-----------------+

1.2 네트워크 구성도
''''''''''''''''''''''''''




2. 초기 셋팅
----------------

    * 아래는 기존에 설치하는것과 거의 동일 합니다.


    .. code-block:: bash

        * Controller Node, Compute Node

        sudo mkdir /opt/stack

        sudo useradd -s /bin/bash -d /opt/stack -m stack
        echo "stack ALL=(ALL) NOPASSWD: ALL" | sudo tee /etc/sudoers.d/stack
        sudo chown stack:stack -R /opt/stack
        sudo -u stack -i

        git clone https://opendev.org/openstack/devstack
        cd devstack
        git checkout stable/wallaby


    .. code-block:: bash

        * Controller Node

        vi /devstack/local.conf
        --------------------------------------

        [[local|localrc]]
        HOST_IP=172.16.0.11
        SERVICE_HOST=$HOST_IP
        MYSQL_HOST=$SERVICE_HOST
        RABBIT_HOST=$SERVICE_HOST
        GLANCE_HOSTPORT=$SERVICE_HOST:9292
        ADMIN_PASSWORD=secret
        DATABASE_PASSWORD=$ADMIN_PASSWORD
        RABBIT_PASSWORD=$ADMIN_PASSWORD
        SERVICE_PASSWORD=$ADMIN_PASSWORD

        ## Neutron options
        Q_USE_SECGROUP=True
        FLOATING_RANGE="172.16.0.0/24"
        IPV4_ADDRS_SAFE_TO_USE="10.0.0.0/22"
        Q_FLOATING_ALLOCATION_POOL=start=172.16.0.50,end=172.16.0.100
        PUBLIC_NETWORK_GATEWAY="172.16.0.2"
        PUBLIC_INTERFACE=ens34

        # Open vSwitch provider networking configuration
        Q_USE_PROVIDERNET_FOR_PUBLIC=True
        OVS_PHYSICAL_BRIDGE=br-ex
        PUBLIC_BRIDGE=br-ex
        OVS_BRIDGE_MAPPINGS=public:br-ex

    .. code-block:: bash

        * Controller Node
        ./stack

--------

    .. code-block:: bash

        * Compute Node

        vi /devstack/local.conf
        --------------------------------------

        [[local|localrc]]
        HOST_IP=172.16.0.12
        SERVICE_HOST=172.16.0.11
        MYSQL_HOST=$SERVICE_HOST
        RABBIT_HOST=$SERVICE_HOST
        GLANCE_HOSTPORT=$SERVICE_HOST:9292

        ADMIN_PASSWORD=secret
        DATABASE_PASSWORD=$ADMIN_PASSWORD
        RABBIT_PASSWORD=$ADMIN_PASSWORD
        SERVICE_PASSWORD=$ADMIN_PASSWORD
        DATABASE_TYPE=mysql

        FIXED_RANGE="10.0.0.0/22"
        FLOATING_RANGE="172.16.0.0/24"

        LOGFILE=/opt/stack/logs/stack.sh.log
        ENABLED_SERVICES=n-cpu,q-agt,c-vol,placement-client

        NOVA_VNC_ENABLED=True
        NOVNCPROXY_URL="http://$SERVICE_HOST:6080/vnc_lite.html"
        VNCSERVER_LISTEN=$HOST_IP
        VNCSERVER_PROXYCLIENT_ADDRESS=$VNCSERVER_LISTEN

    .. code-block:: bash

        * Compute Node
        ./stack


--------

    * Controller Node와 Compute Node의 설치가 완료되면
      Controller Node에서 아래 명령어를 입력합니다.

    .. code-block:: bash

        * Controller Node

        nova service-list --binary nova-compute     # Compute Node가 잘 추가가 되었는지 확인

        ./tools/discover_hosts.sh   # Compute Node를 단일 셀에 매핑

        openstack hypervisor list   # 조금만 기다리면 추가된 Compute Node 확인
