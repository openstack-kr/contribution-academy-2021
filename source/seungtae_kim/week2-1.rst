==========================================================
OpenStack 팀 2주차 진행사항 : CLI와 친해지기
==========================================================

2주차 과제의 모든 진행은 아래와 같은 순서로 차근차근 실행했습니다.

아직 오픈 스택을 배워가는 단계라 GUI, 터미널, 그리고 파이참의 CLI를 사용해 각각 커맨드를 입력해 과제를 수행했습니다.

다만, 4번째 과제 옵션 사항은 해당 네트워크가 private network같은데 demo 프로젝트로 변경하는 방법을 구글링하다가 못찾아서 아직 수행하지 못했습니다.

1. OpenStack GUI 상에서 과제 사항을 실행해보기

2. SSH를 사용해 접속 후, 터미널로 실행 (터미널 명령어는 파이참의 CLI에서 동일한 결과를 출력할 것이라 가정)

3. GUI에서 실패한 사항을 찾아 수정 (동일한 에러가 CLI 상에서도 발생할 것이 분명하기 때문에)

4. OpenStack CLI를 사용해서 실행

1. cirros image로 인스턴스 생성을 cli로 해보기
-----------------------------------------------

사용한 명령어 :

`> openstack server create --flavor <flavor name> --image <image file name> --nic net-id=<network-provider id> <instance name>`

2번에서 수행할 우분투 이미지 파일을 다운로드 받아서 이미지로 등록한 후, 위와 같은 명령어를 수행해서 인스턴스를 만들었습니다.

.. image:: https://user-images.githubusercontent.com/40455392/130324670-c4669df1-0533-4606-8693-6c580e694326.png
   :width: 1000px

.. image:: https://user-images.githubusercontent.com/40455392/130324772-c49ec68e-8493-411e-96f6-b196dd3ff85a.png
   :width: 1000px



2. ubuntu 이미지를 받고, root password를 설정한 다음 cli로 이미지 등록한 후 인스턴스 생성하고 접속까지 하기
------------------------------------------------------------------------------------------------------------

2-1) 우분투 이미지 파일 다운로드 : `wget https://cloud-images.ubuntu.com/focal/current/focal-server-cloudimg-amd64.img`

2-2) 이미지 파일 등록하기

    2-2-1) 이미지 파일을 등록하는 단계에서 아래와 같은 방법으로 이미지 파일 비밀번호 초기화를 시도했고, 성공하지 못했습니다.
        - `sudo virt-customize -a focal-server-cloudimg-amd64.img --root-password password:secret`

    .. image:: https://user-images.githubusercontent.com/40455392/130325196-e8f0e475-6a71-4ff0-a040-e8c6876004dc.png
       :width: 1000px


    2-2-2) 이에 따라 해당 이미지 파일로 우분투 인스턴스 생성 시, 비밀번호 설정이 안되는데 이미지 파일에 비밀번호는 걸린 상황이라 위에 설정한 이미지 파일 비밀번호를 삭제해야하는 상황이 발생했습니다.
        - `sudo virt-customize -a focal-server-cloudimg-amd64.img --root-password password:secret --uninstall cloud-init`

    2-2-3) 결국에 이미지 파일을 등록하는 시점에 비밀번호를 설정하는 방법을 못찾아서, 아래와 같이 `userdata.txt` 를 사용해서 커맨드에 입력하면 비밀번호가 설정되는 것을 확인할 수 있었습니다.

    .. code-block:: 

       #cloud-config
       password: secret
       chpasswd: { expire: False }
       ssh_pwauth: True

    위 텍스트 파일을 설정 후, 인스턴스 생성 명령어에 아래와 같은 옵션을 추가하면 비밀번호 설정이 가능합니다.

    .. code-block::

        > openstack server create --flavor m1.small --image ubuntu --nic net-id=c3262e50-e577-4091-8ac2-f006380397bb --security-group 65febf57-ca19-44a9-8518-e942c2ca7769 --user-data=userdata.txt my-ubuntu

    마찬가지로 위 설정은 GUI에서 그대로 사용 가능합니다.

    .. image:: https://user-images.githubusercontent.com/40455392/130325249-b1c42b57-2ff8-409d-b9aa-41f885ae56e0.png
        :width: 1000px

    2-2-4) 그리고 우분투 로그인을 콘솔에서 할 수 있는 것을 확인했습니다. 그러나 해당 인스턴스로 원격 접속이 안되서 아직 확인 중에 있습니다.

    .. image:: https://user-images.githubusercontent.com/40455392/130325369-995cf415-7277-4f54-97be-d22931a0e5be.png
        :width: 1000px


3.  cli로 floating ip 생성 후 인스턴스에 할당 / 해제 해보기
------------------------------------------------------------

    3-1) CMD 명령어 찾기

        - `> openstack floating ip create public` 를 실행해 admin 프로젝트에서 floating ip를 생성했습니다.

        .. image:: https://user-images.githubusercontent.com/40455392/130325507-2edc1301-4d31-4753-a0e5-40f807d6c0e2.png
            :width: 1000px

        - 다만, Cirros에서는 실행이 문제 없이 되었으나, Ubuntu에서 네트워크를 Public으로 작업 시, 사용이 안된다는 것을 확인했고 (아마 floating ip를 private or shared로 해야 사용 가능한)

        - 새로 인스턴스를 만들면서 인스턴스의 네트워크를 shared로 설정해 ip를 할당 / 해제하는 실습을 하게 되었습니다.

    3-2) shared network에 대한 라우터 설정하기

        3-2-1) shared network를 사용해 우분투 인스턴스를 생성하니, 바로 floating ip를 설정할 수 없었습니다.

        이에 따라 public 라우터를 생성해 해당 서브넷에 shared network로 생성된 인스턴스의 ip를 등록해야 floating ip를 할당할 수 있었습니다.

        .. image:: https://user-images.githubusercontent.com/40455392/130325648-1ade1842-d920-4cef-a1bd-a59d54c1d1cd.png
            :width: 1000px

        3-2-2) 본격 ip 할당 / 해제하기

        설정이 어렵지, 막상 입력하는 명령어는 매우 쉬웠습니다.

        ip 할당하기 : `server add floating ip <instance name> <floating ip> (ip 할당하기 명령어)`

        .. image:: https://user-images.githubusercontent.com/40455392/130325553-84792817-83a1-40ea-8dfd-c44189aa356a.png
            :width: 1000px

        ip 해제하기 : `server remove floating ip <instance name> <floating ip> (ip 해제하기 명령어)`

        .. image:: https://user-images.githubusercontent.com/40455392/130325557-ab138022-ce2f-4715-aa45-2a2f8e66476b.png
            :width: 1000px


4. 10.8.0.0/24 네트워크를 만들고 public network와 연결하는 과정을 cli로 해보기  (optional)
-------------------------------------------------------------------------------------------

    4-1) 이 작업이 네트워크를 private으로 floating ip를 생성하고 public network와 연결하는 작업을 의미하는 걸까요?

    4-2) 해당 네트워크를 설정하는 작업에 대해 구글링을 어떻게 할 지 키워드를 몰라서 문의드립니다.

5. Error 처리
-----------------

    5-1) 작업하면서 워낙 인스턴스를 많이 생성 / 삭제하다보니 아래와 같은 에러명을 자주 목격했습니다.

        - Exhausted all hosts available for retrying build failures for instance

        - 해결방법은 배포한 OpenStack 내의 리소스가 부족하다는 명령어인데, 다른 리소스들은 풍족한데 하이퍼바이저 메뉴의 VCPU의 최대 용량이 작아 발생한 에러였습니다.

        - 생성한 인스턴스의 거의 대부분을 정리하고, 볼륨에 남아있던 값들 중 삭제 안한 인스턴스와 관련된 것을 제외하고 모두 지우니 해당 에러가 해결되었습니다.


6. Reference
--------------------------

- `우분투 인스턴스 비밀번호 설정하기 <https://techglimpse.com/nova-boot-instance-with-password/>`_

- `shared network 라우터 연결하기 <https://github.com/AJNOURI/COA/issues/64>`_

- `floating ip 할당/해제하기 <https://docs.openstack.org/ocata/user-guide/cli-manage-ip-addresses.html>`_

