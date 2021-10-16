===============================================
3week - openstack server list에 field 추가하기
===============================================

--------
Homework
--------

openstack server list 에 field 추가하기
openstack server list 를 하면 기본적으로 "ID", "Name", "Status", "Networks" ,"Image", "Flavor" 필드만 보입니다.

.. code-block:: text

    +--------------------------------------+------+--------+--------------------------------------------------------------------------+--------------------------+---------+
    | ID                                   | Name | Status | Networks                                                                 | Image                    | Flavor  |
    +--------------------------------------+------+--------+--------------------------------------------------------------------------+--------------------------+---------+
    | 3d35741b-0d50-41d7-b828-7740a6b322f6 | demo | ACTIVE | private=10.0.0.15, 192.168.100.116, fdc4:5c26:6ff9:0:f816:3eff:fe41:c156 | N/A (booted from volume) | m1.nano |
    +--------------------------------------+------+--------+--------------------------------------------------------------------------+--------------------------+---------+

사실 서버 정보는 위 6개 필드 말고도 더 많이 있습니다.

.. code-block:: text

    +-------------------------------------+--------------------------------------------------------------------------+
    | Field                               | Value                                                                    |
    +-------------------------------------+--------------------------------------------------------------------------+
    | OS-DCF:diskConfig                   | AUTO                                                                     |
    | OS-EXT-AZ:availability_zone         | nova                                                                     |
    | OS-EXT-SRV-ATTR:host                | pyk-large                                                                |
    | OS-EXT-SRV-ATTR:hypervisor_hostname | pyk-large                                                                |
    | OS-EXT-SRV-ATTR:instance_name       | instance-00000001                                                        |
    | OS-EXT-STS:power_state              | Running                                                                  |
    | OS-EXT-STS:task_state               | None                                                                     |
    | OS-EXT-STS:vm_state                 | active                                                                   |
    | OS-SRV-USG:launched_at              | 2021-08-11T10:39:42.000000                                               |
    | OS-SRV-USG:terminated_at            | None                                                                     |
    | accessIPv4                          |                                                                          |
    | accessIPv6                          |                                                                          |
    | addresses                           | private=10.0.0.15, 192.168.100.116, fdc4:5c26:6ff9:0:f816:3eff:fe41:c156 |
    | config_drive                        |                                                                          |
    | created                             | 2021-08-11T10:39:16Z                                                     |
    | flavor                              | m1.nano (42)                                                             |
    | hostId                              | 032bc4d449912d9557714f16d432b1babe659d44b6fc47ef4e01b108                 |
    | id                                  | 3d35741b-0d50-41d7-b828-7740a6b322f6                                     |
    | image                               | N/A (booted from volume)                                                 |
    | key_name                            | None                                                                     |
    | name                                | demo                                                                     |
    | progress                            | 0                                                                        |
    | project_id                          | b444a96b37ce4dfdac099ada35d13c48                                         |
    | properties                          |                                                                          |
    | security_groups                     | name='default'                                                           |
    | status                              | ACTIVE                                                                   |
    | updated                             | 2021-08-11T10:39:42Z                                                     |
    | user_id                             | ac2efc75e06f4497b1c9a3d2550fffb5                                         |
    | volumes_attached                    | id='279aaf00-54d6-4cf6-8e08-d0937bf081ac'                                |
    +-------------------------------------+--------------------------------------------------------------------------+

미션1: openstack server list 의 기본 결과 필드에 "Project ID" 를 추가하기

미션2: openstack server list 의 기본 결과 필드에 "Created At"를 추가하기

------------------------
Background Knowledge
------------------------

이전 과제를 통해서 실제로 api를 호출해서 데이터를 튜플형태로 만드는 :code:`server.ListServer.take_action` 함수와 테이블을 출력하기 위해 모양을 이쁘게 만드는 작업을 하는 :code:`table.TableFormatter.emit_list` 함수를 알아낼 수 있었다.

해당 문제는 테이블 출력 자체를 수정하는 것이 아니라 테이블에 추가할 자료를 추가하는 것이므로 :code:`server.ListServer.take_action` 부분을 중점적으로 봐야되지 않을까 생각해볼 수 있다.


미션1 & 미션2 : openstack server list 의 기본 결과 필드에 "Project ID", "Created At" 를 추가하기
===============================================================================================================

현재 :code:`server list` 의 실행 결과 테이블 컬럼은 id, name, status, networks, image, flavor 이렇게 6가지가 있다.

이 전 과제에서 :code:`server.ListServer.take_action` 는 테이블을 그리기 위한 :code:`tuple` 형태의 데이터를 만드는데, 데이터를 설정하는 부분이 딱 봐도 아래 부분과 연관된 것을 눈치 껏 알 수 있다.

.. code-block:: python

    else:
        columns = (
            'ID',
            'Name',
            'Status',
            'Networks',
            'Image Name',
            'Flavor Name',
        )
    column_headers = (
        'ID',
        'Name',
        'Status',
        'Networks',
        'Image',
        'Flavor',
    )

해당 부분에 아래와 같이 'tenant_id', 'created', 'Project ID', 'Created At' 항목을 넣으면 원하는 결과를 얻을 수 있다.

.. code-block:: python

    else:
        columns = (
            'tenant_id',
            'ID',
            'Name',
            'Status',
            'Networks',
            'Image Name',
            'Flavor Name',
            'created',
        )
    column_headers = (
        'Project ID',
        'ID',
        'Name',
        'Status',
        'Networks',
        'Image',
        'Flavor',
        'Created At',
    )

.. code-block:: text

    +----------------------------------+--------------------------------------+--------------+--------+--------------------------------------------------------------------------+------------------------------------------------------+---------+----------------------+
    | Project ID                       | ID                                   | Name         | Status | Networks                                                                 | Image                                                | Flavor  | Created At           |
    +----------------------------------+--------------------------------------+--------------+--------+--------------------------------------------------------------------------+------------------------------------------------------+---------+----------------------+
    | 1da00946e13541b196b7890c8ace8177 | 2523f0cb-a140-4571-8e41-7efe988e22c8 | network-test | ACTIVE | private_10.8.0.0=10.8.0.223                                              | N/A (booted from volume)                             | m1.nano | 2021-08-29T15:01:03Z |
    | 1da00946e13541b196b7890c8ace8177 | e8dc1163-8bcd-49a0-9e27-8d7b9c6a3abd | ubuntu20     | ACTIVE | private=10.0.0.32, 192.168.100.163, fd77:ff60:4ca8:0:f816:3eff:fe20:3e77 | ubuntu.focal-server-cloudimg-amd64-root-pw-is-secret | ds512M  | 2021-08-29T12:47:22Z |
    +----------------------------------+--------------------------------------+--------------+--------+--------------------------------------------------------------------------+------------------------------------------------------+---------+----------------------+

이 방법은 두 가지로 생각해 낼 수 있다.

한 가지 방법은 해당 코드 바로 아래에 

.. code-block:: python
    
    # support for additional columns
    if parsed_args.columns:
        # convert tuple to list to edit them
        column_headers = list(column_headers)
        columns = list(columns)

        for c in parsed_args.columns:
            if c in ('Project ID', 'project_id'):
                columns.append('tenant_id')
                column_headers.append('Project ID')
            if c in ('User ID', 'user_id'):
                columns.append('user_id')
                column_headers.append('User ID')
            if c in ('Created At', 'created_at'):
                columns.append('created_at')
                column_headers.append('Created At')

이 부분을 보면 어떤 조건에 의해서 추가 컬럼을 세팅하는 코드를 볼 수 있는데 여기서 위에 파악했던 데이터 구조에 project id, created at를 넣기 위한 데이터를 넣는 것을 볼 수 있다.

다른 나머지 방법은 아래 데이터가 세팅되는 과정을 분석하는 것이다.

아래 코드의 플로우를 보면 크게 현재 없는 flavor, image 정보 데이터를 채우기위해 관련 api를 호출해서 정보를 가져와서 동일한 형태의 데이터 구조를 구성하고, 테이블의 데이터를 채워넣는데 원본 데이터의 키값과 현재 세팅한(위에 :code:`columns` 에 기재했던 값) string이 다르기 때문에 일련의 규칙을 사용해 맞춰주는 작업을 :code:`utils.get_item_properties` 에서 한다.

해당 함수를 간단하게 보면 빈칸을 언더스코어로 변경하는 등의 동작을 하면서 실제 api를 통해 받은 json의 키와 매칭시키기 위한 변환 작업을 수행한다.


----------------
Reference
----------------
