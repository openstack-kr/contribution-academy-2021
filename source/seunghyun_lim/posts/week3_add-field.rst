Week 3. openstack server list 결과 테이블에 field 추가하기
==========================================================

3주차 과제 풀이입니다. 동작 원리를 파악하여 필드를 추가하는 것이 핵심이었습니다.

- 미션 1. openstack server list 의 기본 결과 필드에 "Project ID" 를 추가하기
- 미션 2. openstack server list 의 기본 결과 필드에 "Created At"를 추가하기

목차
----------
1. `풀이 방법`_
2. `원리`_
3. `버그 발견`_

풀이 방법
------------------------------------
- 표의 컨텐츠를 선별하는 ListServer의 Column에 필드 변수를 추가합니다.

.. code-block:: python

    # python-openstackclint/openstackclient/compute/v2/server.py/ListServer.take_action()
    columns = (
        'ID',
        'tenant_id', # project ID의 값
        'Name',
        'Status',
        'Networks',
        'Image Name',
        'Flavor Name',
        'created', # Create At의 값
    )
    column_headers = (
        'ID',
        'Project ID', # project ID의 행 이름
        'Name',
        'Status',
        'Networks',
        'Image',
        'Flavor',
        'Created At', # Create At의 행 이름
    )

|

결과
~~~~~~~~~~~~~~

.. code-block:: bash

    +--------------------------------------+----------------------------------+--------+--------+---------------------+--------------+----------+----------------------+
    | ID                                   | Project ID                       | Name   | Status | Networks            | Image        | Flavor   | Created At           |
    +--------------------------------------+----------------------------------+--------+--------+---------------------+--------------+----------+----------------------+
    | xxxxx-xxxxx-xxxxxx-xxxxxx-xxxxxxxxxx | xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx | ubuntu | ACTIVE | private=xx.xx.xx.xx | ubuntu-20.04 | m1.small | 2021-08-22T18:54:02Z |
    +--------------------------------------+----------------------------------+--------+--------+---------------------+--------------+----------+----------------------+

    Process finished with exit code 0

-------------------------

원리
------------------
- ListServer의 take_action 함수의 return 부분을 보면 table라는 튜플 객체에 값을 열람할 수 있게 구성하여 리턴합니다.
- 이 때 table의 열 이름을 결정 짓는 `column_headers` 변수와 열 이름에 따라 찾을 값 이름을 가지는 `columns` 변수는 take_action에서 선언됩니다.

.. code-block:: python

    # python-openstackclint/openstackclient/compute/v2/server.py
    class ListServer(command.Lister):
        def take_action(self, parsed_args):
            ...
            table = (
                column_headers, # 열 이름 결정
                (
                    utils.get_item_properties(
                        s, columns, # columns : 열 이름에 따라 참조할 값 이름
                        mixed_case_fields=mixed_case_fields,
                        formatters={
                            'OS-EXT-STS:power_state': PowerStateColumn,
                            'Networks': format_columns.DictListColumn,
                            'Metadata': format_columns.DictColumn,
                        },
                    ) for s in data # columns가 참조할 server의 정보를 가지고 있다.
                ),
            )
            return table

|

- 이 후 table을 구성할 때, data 변수 속에 server의 정보를 담고 있는 search_opts의 key 값 중 columns에 해당되는 값이 존재할 시 값을 출력하게 됩니다.

.. code-block:: python

    # python-openstackclint/openstackclient/compute/v2/server.py/ListServer.take_action()
    search_opts = {
            'reservation_id': parsed_args.reservation_id,
            'ip': parsed_args.ip,
            'ip6': parsed_args.ip6,
            'name': parsed_args.name,
            'instance_name': parsed_args.instance_name,
            'status': parsed_args.status,
            'flavor': flavor_id,
            'image': image_id,
            'host': parsed_args.host,
            'tenant_id': project_id,
            'all_tenants': parsed_args.all_projects,
            'user_id': user_id,
            'deleted': parsed_args.deleted,
            'changes-before': parsed_args.changes_before,
            'changes-since': parsed_args.changes_since,
        }
    ...

    data = compute_client.servers.list(search_opts=search_opts,
                                           marker=marker_id,
                                           limit=parsed_args.limit)

|

버그 발견
--------------------
- Create At을 필드에 추가하던 중 팀원인 이재용님이 columns (server 정보 데이터의 값 이름)과 nova에서 받아오는 server data 값 이름이 달라 충돌하는 버그를 발견했다.
- 실제로 wallaby 버전과 master 버전에서 server list에 특정 columns을 추가하는 -c 옵션을 사용해서 created at 생성 날짜를 추가하면 에러가 난다.

.. code-block:: bash

    $ ~/devstack> openstack server list -c created_at
        No recognized column names in ['create_at']. Recognized columns are ('ID', 'Project ID', 'Name', 'Status', 'Networks', 'Image', 'Flavor', 'Created At').

- 이는 아래와 같이 columns 값을 받아오는 이름이 created_at인데 data의 search_opt에서는 created으로 받아오게 되어 문제가 생긴다.

.. code-block:: python

    # python-openstackclint/openstackclient/compute/v2/server.py/ListServer.take_action()
    # support for additional columns
    if parsed_args.columns:
        for c in parsed_args.columns:
            if c in ('Created At', 'created_at'):
                columns.append('created_at')
                column_headers.append('Created At')


- 끝으로 PR 링크를 첨부하고 마치겠다.
- server list의 Create At, Bug PR : https://review.opendev.org/c/openstack/python-openstackclient/+/806464