==========================================================
클라우드 가상 머신에서 devstack 설치하기
==========================================================

본 글에서는 클라우드 가상 머신에서 DevStack 을 설치하는 방법에 대해서 다룹니다.

추가 디스크를 연결하지 않았다면, (Optinal)은 생략합니다.


**오픈스택을 리눅스 환경에서 자동으로 구축해주는 DevStack**
 - DevStack 은 빠르고 쉽게 오픈스택의 핵심 컴포넌트를 구축하기 위한 스크립트입니다.
 - OpenStack 프로젝트의 다양한 기능을 테스트하기 위한 용도로도 사용됩니다.

|

주요 컴포넌트
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
**DevStack을 이용하면 이 컴포넌트들을 쉽게 구성할 수 있습니다.**
 - **Nova**: 가상의 서버를 생성하기 위한 Nova
 - **Glance**: 서버의 기동 이미지를 다루는 Glance
 - **Cinder**: 블록 스토리지를 제어하기 위한 Cinder
 - **Neutron**: 가상의 네트워크를 만들기 위한 Neutron
 - **Keystone**: 사용자에 대한 인증을 처리하는 Keystone
 - **Placement**: 자원을 관리하기 위한 Placement
 - **Horizon DashBoard**: Horizon 대시보드를 통해 GUI 환경에서 오픈스택의 핵심 컴포넌트 API를 컨트롤할 수 있습니다.

|

DevStack 지원 Linux OS
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
 - Ubuntu
 - CentOS
 - OpenSUSE

|

(Optional) 클라우드 환경 블록 스토리지 마운트
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

추가한 블록 스토리지 디스크를 **/opt/stack 경로에 마운트** 하여 진행합니다.
 - 추가 디스크 정보 확인

 .. code-block:: none
    :emphasize-lines: 7

     ubuntu@doa-wallaby:~$ lsblk
     NAME    MAJ:MIN RM  SIZE RO TYPE MOUNTPOINT
     vda     252:0    0   30G  0 disk
     ├─vda1  252:1    0 29.9G  0 part /
     ├─vda14 252:14   0    4M  0 part
     └─vda15 252:15   0  106M  0 part /boot/efi
     vdb     252:16   0   50G  0 disk => 연결된 Block Storage

|

 - vdb 디바이스를 xfs 파일 시스템으로 포맷합니다.

 .. code-block:: none

    ubuntu@doa-wallaby:~$ sudo mkfs.xfs -f -i size=1024 /dev/vdb
    meta-data=/dev/vdb               isize=1024   agcount=4, agsize=3276800 blks
             =                       sectsz=512   attr=2, projid32bit=1
             =                       crc=1        finobt=1, sparse=0, rmapbt=0, reflink=0
    data     =                       bsize=4096   blocks=13107200, imaxpct=25
             =                       sunit=0      swidth=0 blks
    naming   =version 2              bsize=4096   ascii-ci=0 ftype=1
    log      =internal log           bsize=4096   blocks=6400, version=2
             =                       sectsz=512   sunit=0 blks, lazy-count=1
    realtime =none                   extsz=4096   blocks=0, rtextents=0

|

 - 다음과 같은 명령으로 /opt/stack 경로에 마운트 합니다.

 .. code-block:: none

    sudo mount -t xfs /dev/vdb /opt/stack

|

 - 마운트 및 파일 시스템 포맷이 올바르게 되었는지 확인합니다.

 .. code-block:: none
    :emphasize-lines: 11

    ubuntu@doa-wallaby:~$ df -hT
    Filesystem     Type      Size  Used Avail Use% Mounted on
    udev           devtmpfs  3.9G     0  3.9G   0% /dev
    tmpfs          tmpfs     798M  700K  797M   1% /run
    /dev/vda1      ext4       29G  1.5G   28G   6% /
    tmpfs          tmpfs     3.9G     0  3.9G   0% /dev/shm
    tmpfs          tmpfs     5.0M     0  5.0M   0% /run/lock
    tmpfs          tmpfs     3.9G     0  3.9G   0% /sys/fs/cgroup
    /dev/vda15     vfat      105M  6.6M   98M   7% /boot/efi
    tmpfs          tmpfs     798M     0  798M   0% /run/user/1000
    /dev/vdb       xfs        50G   58M   50G   1% /opt/stack

|

사용자 계정 생성
~~~~~~~~~~~~~~~~~~~~~~~~

 - 다음과 같은 명령으로 stack 사용자 계정을 생성합니다.

 .. code-block:: none

    sudo useradd -s /bin/bash -d /opt/stack -m stack

|

 - 생성된 사용자 계정을 확인합니다.
 - 사용자 계정은 /etc 아래의 passwd 파일에서 확인이 가능합니다.

 .. code-block:: none

    ubuntu@doa-wallaby:~$ cat /etc/passwd | grep stack
    stack:x:1001:1001::/opt/stack:/bin/bash

|

 - 생성한 stack 계정 권한으로 /opt/stack 경로 내의 모든 디렉터리 소유주와 소유 그룹을 변경합니다.

 .. code-block:: none

    sudo chown stack:stack -R /opt/stack/

|

 - 생성한 사용자의 sudo 권한을 부여합니다.
 - /etc 아래의 sudoers.d 디렉터리 내에 파일을 추가하여 특정 사용자에 대한 sudo 권한을 부여할 수 있습니다.

 .. code-block:: none

    echo "stack ALL=(ALL) NOPASSWD: ALL" | sudo tee /etc/sudoers.d/stack
    sudo su - stack

|

데브스택 가져오기
~~~~~~~~~~~~~~~~~~~~~~~

 - 현재일 기준(2021-10) 안정적인 버전인 wallaby 로 설치를 진행합니다.
 - 설치를 코드블록의 링크에서 DevStack의 소스를 다운로드 받습니다.

 .. code-block:: none

    git clone https://opendev.org/openstack/devstack
    cd devstack

|

 - 작업 브랜치를 stable/wallaby 브랜치로 checkout 을 받아 진행합니다.

 .. code-block:: none

    git checkout stable/wallaby

|

local.conf 설정 및 가상의 public network 설정
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

 - bridge-utils 패키지를 이용하여 가상의 프로바이더 네트워크를 구성합니다.
 - 이 패키지로 네트워크 인터페이스 카드 여러개를 논리적으로 하나로 묶어, 브릿지를 만들 수 있습니다.
 - 이에 대한 내용은 `멘토님의 블로그 <https://printf.kr/14>`_ 에서도 확인할 수 있습니다.

 .. code-block:: none

    sudo apt install bridge-utils

|

 - 아래 명령으로 mybr0 브릿지를 생성합니다.

 .. code-block:: none

    sudo brctl addbr mybr0

|

 - 생성한 mybr0 브릿지 인터페이스를 확인하기 위해 네트워크 인터페이스 정보를 확인합니다.
 - 네트워크 인터페이스 정보를 확인하기 위한 명령어로는 ifconfig 혹은 ip addr 로 확인이 가능합니다.

 .. code-block:: none
    :emphasize-lines: 14,15

    stack@doa-wallaby:~/devstack$ ip add
    1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
        link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
        inet 127.0.0.1/8 scope host lo
           valid_lft forever preferred_lft forever
        inet6 ::1/128 scope host
           valid_lft forever preferred_lft forever
    2: ens3: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1450 qdisc fq_codel state UP group default qlen 1000
        link/ether fa:16:3e:3b:ad:5b brd ff:ff:ff:ff:ff:ff
        inet 192.168.1.55/24 brd 192.168.1.255 scope global dynamic ens3
           valid_lft 47327sec preferred_lft 47327sec
        inet6 fe80::f816:3eff:fe3b:ad5b/64 scope link
           valid_lft forever preferred_lft forever
    3: mybr0: <BROADCAST,MULTICAST> mtu 1500 qdisc noop state DOWN group default qlen 1000
        link/ether 36:74:c0:01:da:12 brd ff:ff:ff:ff:ff:ff

|

 - 생성한 mybr0 브릿지 인터페이스에 192.168.100.1 아이피를 할당하고 서브넷은 255.255.255.0 으로 초기화를 한 후, 링크 상태를 활성화합니다.

 .. code-block:: none

    sudo ifconfig mybr0 192.168.100.1 netmask 255.255.255.0 up

|

 - iptables 를 이용해 패킷이 라우터로 사용되는 호스트를 통과하도록 Forward Chain 정책을 정의하여 패킷 포워딩 규칙을 설정합니다.

 .. code-block:: none

    sudo iptables -I FORWARD -j ACCEPT

|

 - 정의한 Chain 정책을 확인합니다.

 .. code-block:: none
    :emphasize-lines: 7

    stack@doa-wallaby:~/devstack$ sudo iptables -nL
    Chain INPUT (policy ACCEPT)
    target     prot opt source               destination

    Chain FORWARD (policy ACCEPT)
    target     prot opt source               destination
    ACCEPT     all  --  0.0.0.0/0            0.0.0.0/0

    Chain OUTPUT (policy ACCEPT)
    target     prot opt source               destination

|

 - mybr0 인터페이스에 할당된 사설 아이피에 대한 패킷을 postrouting 체인을 선언해 외부 네트워크에서 액세스가 가능하도록 NAT 규칙을 설정을 합니다.

 .. code-block:: none

    sudo iptables -t nat -I POSTROUTING -s 192.168.100.0/24 -j MASQUERADE

|

 - 선언한 체인 정책을 확인합니다.

 .. code-block:: none
    :emphasize-lines: 13

    stack@doa-wallaby:~/devstack$ sudo iptables -t nat -nL
    Chain PREROUTING (policy ACCEPT)
    target     prot opt source               destination

    Chain INPUT (policy ACCEPT)
    target     prot opt source               destination

    Chain OUTPUT (policy ACCEPT)
    target     prot opt source               destination

    Chain POSTROUTING (policy ACCEPT)
    target     prot opt source               destination
    MASQUERADE  all  --  192.168.100.0/24     0.0.0.0/0

|

 - DevStack 을 설치하면 기본적으로 OS에 설정된 아이피로 Horizon 을 비롯한 API endpoint 가 지정됩니다.
 - 클라우드 인스턴스에 설정된 아이피는 사설 아이피이므로 DevStack 설정 파일인 local.conf 파일에 외부 통신이 가능한 인스턴스의 공인 아이피 정보를 넣어주고, 그 아이피로 바인딩 할 수 있도록 루프백 인터페이스에 아이피를 설정합니다.
 - 루프백 인터페이스에 대한 바인딩 설정은 아래와 같이 진행합니다.

 .. code-block:: none

    sudo ip addr add 211.37.149.100/32 dev lo

|

 - ip addr 혹은 ifconfig 와 같은 명령어로 루프백 인터페이스에 바인딩된 정보를 확인합니다.

 .. code-block:: none
    :emphasize-lines: 6

    stack@doa-wallaby:~/devstack$ ip add
    1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
        link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
        inet 127.0.0.1/8 scope host lo
           valid_lft forever preferred_lft forever
        inet 211.37.149.100/32 scope global lo
           valid_lft forever preferred_lft forever
        inet6 ::1/128 scope host
           valid_lft forever preferred_lft forever

|

local.conf 설정
~~~~~~~~~~~~~~~~~~~~~~

 - 모든 네트워크 설정이 완료되었으므로, local.conf 파일을 수정하여 스크립트를 실행합니다.
 - 데브스택의 구성은 local.conf 파일을 통해 설정할 수 있습니다.
 - 이 파일의 샘플은 devstack/samples 경로에 위치하며, DevStack 설치를 진행할 때에는 /devstack 디렉터리에서 local.conf 파일을 생성하여 진행하여야 합니다.

 .. code-block:: none

    stack@doa-wallaby:~/devstack$ pwd               # local.conf 파일 설정 경로
    /opt/stack/devstack

    stack@doa-wallaby:~/devstack$ vim local.conf    # 편집기를 이용해 local.conf 파일을 설정합니다.

|

 - local.conf 설정은 openswitch(OVS)를 이용하여 네트워크를 구성하며, floating ip 로는 192.168.100.0/24 를 사용합니다. 네트워크를 제공하는 인터페이스는 mybr0 리눅스 브릿지로 지정합니다.
 - **HOST_IP**: 컴퓨트 노드의 IP, 현재 서버의 공인 아이피
 - **ADMIN_PASSWORD**: 관리자 패스워드 (DATABASE_PASSSWORD, RABBIT_PASSWORD, SERVICE_PASSWORD)
 - **Q_FLOATING_ALLOCATION_POOL**: 오픈스택이 생성하는 인스턴스에 할당할 Floating ip 대역

 .. code-block:: none

    [[local|localrc]]
    HOST_IP=211.37.149.100
    FORCE=yes
    ADMIN_PASSWORD=secret
    DATABASE_PASSWORD=$ADMIN_PASSWORD
    RABBIT_PASSWORD=$ADMIN_PASSWORD
    SERVICE_PASSWORD=$ADMIN_PASSWORD

    disable_service etcd3

    ## Neutron options
    Q_USE_SECGROUP=True
    FLOATING_RANGE="192.168.100.0/24"
    IPV4_ADDRS_SAFE_TO_USE="10.0.0.0/22"
    Q_FLOATING_ALLOCATION_POOL=start=192.168.100.50,end=192.168.100.250
    PUBLIC_NETWORK_GATEWAY="192.168.100.1"
    PUBLIC_INTERFACE=mybr0

    # Open vSwitch provider networking configuration
    Q_USE_PROVIDERNET_FOR_PUBLIC=True
    OVS_PHYSICAL_BRIDGE=br-ex
    PUBLIC_BRIDGE=br-ex
    OVS_BRIDGE_MAPPINGS=public:br-ex

|

 - 모든 설정이 완료되었다면, stack.sh 스크립트를 실행하여 설치를 진행합니다.
 - 구축 시간은 약 15-20분 정도 소요됩니다. 기다리는 동안 DevStack 에 대해서 자세하게 알고 싶다면, 다음의 링크를 참고해주세요.
 - URL: https://docs.openstack.org/devstack/latest/

 .. code-block:: none

    stack@doa-wallaby-2:~/devstack$ ls -al stack.sh         # 스크립트 경로는 /devstack 경로에 위치합니다.
    -rwxrwxr-x 1 stack stack 45000 Oct 15 17:05 stack.sh

    stack@doa-wallaby-2:~/devstack$ ./stack.sh              # 스크립트 실행

|

DevStack 설치 완료
~~~~~~~~~~~~~~~~~~~~~~~

 - 정상적으로 설치가 되었다면, Horizon 대시보드로 접속할 수 있는 정보와 사용자 인증 엔드포인트 URL이 출력됩니다.

 .. image:: ../images/devstack_install_01.png

|

 .. image:: ../images/devstack_install_02.png