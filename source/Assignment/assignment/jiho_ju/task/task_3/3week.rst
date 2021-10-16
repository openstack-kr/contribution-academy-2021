3week - openstack server list 에 field 추가하기
==========================================================

contents
---------------
- `1. openstack server list 의 기본 결과 필드에 \"Project ID\" 를 추가하기`_
- `2. openstack server list 의 기본 결과 필드에 \"Created At\"를 추가하기`_

1. openstack server list 의 기본 결과 필드에 \"Project ID\" 를 추가하기
-----------------------------------------------------------------------------

\"server list\" 인자 값을 처리하는 파일은 openstackclient/compute/v2/server 모듈에서 ListServer 클래스 이다.
ListServer 클래스 take_action 메서드에서 테이블에 출력될 \"columns_header\" 와 data 를 받아올 columns 가 어떻게 구성되는지 확인해보자.

.. code-block:: python

   # openstackclent/compute/v2/server.py ListServer
   def take_action(self, parsed_args):
        ...
        if parsed_args.columns:
        # unmodifiable 한 tuple 에서 list로 바꿔야 값을 추가할 수 있다.
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

        # convert back to tuple
        column_headers = tuple(column_headers)
        columns = tuple(columns)
        ...

take_action 메서드에서 기본적으로 column_headers 와 columns 는 다음과 같이 설정된다.

- column_headers: (\'ID\', \'Name\', \'Status\', \'Networks\', \'Image\', \'Flavor\')
- columns: (\'ID\', \'Name\', \'Status\', \'Networks\', \'Image Name\', \'Flavor Name\')

.. code-block:: python

   # openstackclent/compute/v2/server.py ListServer
   def take_action(self, parsed_args):
        ...
        # 과정 1
        data = compute_client.servers.list(search_opts=search_opts,
                                                   marker=marker_id,
                                                   limit=parsed_args.limit)
        ...
        # 과정 2
        table = (
                column_headers,
                (
                    utils.get_item_properties(
                        s, columns,
                        mixed_case_fields=mixed_case_fields,
                        formatters={
                            'OS-EXT-STS:power_state': PowerStateColumn,
                            'Networks': format_columns.DictListColumn,
                            'Metadata': format_columns.DictColumn,
                        },
                    ) for s in data
                ),
            )
        return table

\"과정 1\" 을 수행 결과 \"server list\" 에 해당하는 인스턴스들과 해당 인스턴스들의 정보들이 data 에 할당된다.
\"과정 2\" 를 통해 columns 튜플 각 값들에 상응하는 인스턴스에 대한 정보들을 추출하여 colum_header 와 함께 table 을 구성한다.

\"과정 2\" 에서 colums 값들과 각 인스턴스가 가진 정보(키:값) 키와 비교해서 같다면 해당 값을 추출하여 테이블에 출력하므로 \"과정 2\" 전에 columns 와 colum_header 튜플에 \"tenant_id\", \"Project ID\" 을 각각 추가하자.

.. code-block:: python

   # openstackclent/compute/v2/server.py ListServer
   def take_action(self, parsed_args):
        ...
        column_headers = list(column_headers)
        columns = list(columns)

        ## server list 기본값에 Project ID 값 추가
        columns.append("tenant_id")
        column_headers.append('Project ID')

        if parsed_args.columns:
            ...

결과
^^^^^^^^

.. code-block:: bash

    +--------------------------------------+----------------+--------+--------------------------------------------------------+--------------------------+---------+----------------------------------+
    | ID                                   | Name           | Status | Networks                                               | Image                    | Flavor  | Project ID                       |
    +--------------------------------------+----------------+--------+--------------------------------------------------------+--------------------------+---------+----------------------------------+
    | 478eac81-48b0-43b0-a2a7-66c249aa19c9 | test2_instance | ACTIVE | private=10.0.0.23, fdfe:f92c:c853:0:f816:3eff:fef0:317 | Ubuntu-18.04             | ds512M  | 72405027628a419f8485eb218a19b726 |
    | 942ebcec-4cd1-4386-99a1-7152e7a7b9be | task2_instance | ACTIVE | public=192.168.100.229, 2001:db8::f1                   | Ubuntu-18.04             | ds512M  | 72405027628a419f8485eb218a19b726 |
    | 86256d63-2d36-4aed-a67d-81e6861f12ec | test_instance  | ACTIVE | public=192.168.100.178, 2001:db8::2c7                  | cirros-0.5.2-x86_64-disk | m1.tiny | 72405027628a419f8485eb218a19b726 |
    | ec2c6265-3d0a-4ed2-81c2-7a5f748e9d8f | task1_instance | ACTIVE | public=192.168.100.144, 2001:db8::122                  | cirros-0.5.2-x86_64-disk | m1.tiny | 72405027628a419f8485eb218a19b726 |
    +--------------------------------------+----------------+--------+--------------------------------------------------------+--------------------------+---------+----------------------------------+

columns 에는 왜 \"project_id\" 가 아니라 \"tenant_id\" 를 추가해주나요??
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

\"project\" 와 \"tenant\" 는 같은 개념으로 보면 된다.
keystone이 v2 에서 tenant로 갖고 있던 개념을 v3에서 project로 치환했다.
domain, user group를 추가해서 좀더 자세하게 관리하기 위해서 용어도 이에 맞는 tenant에서 project르 변경한 거 같다.

**결론은 \"project\", \"tenant\" 는 같은 개념이다.**

2. openstack server list 의 기본 결과 필드에 \"Created At\"를 추가하기
------------------------------------------------------------------------------

1 번 문제와 동일한 방법으로 columns, column_header 튜플에 다음과 같은 값을 추가하면 된다.

- colums 튜플에 "created_at" 추가
- column_header 튜플에 "Created At" 추가

.. code-block:: python

   # openstackclent/compute/v2/server.py ListServer
   def take_action(self, parsed_args):
       ...
       # unmodifiable 한 tuple 에서 list로 바꿔야 값을 추가할 수 있다.
       column_headers = list(column_headers)
       columns = list(columns)

       ### server list 기본값에 Created At 값 추가
       columns.append('created')
       column_headers.append('Created At')

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
           columns.append('created_at') # Issue
           column_headers.append('Created At')

       # convert back to tuple
       column_headers = tuple(column_headers)
       columns = tuple(columns)
       ...

결과
^^^^^^^^

.. code-block:: bash

    +--------------------------------------+----------------+--------+--------------------------------------------------------+--------------------------+---------+----------------------+
    | ID                                   | Name           | Status | Networks                                               | Image                    | Flavor  | Created At           |
    +--------------------------------------+----------------+--------+--------------------------------------------------------+--------------------------+---------+----------------------+
    | 478eac81-48b0-43b0-a2a7-66c249aa19c9 | test2_instance | ACTIVE | private=10.0.0.23, fdfe:f92c:c853:0:f816:3eff:fef0:317 | Ubuntu-18.04             | ds512M  | 2021-08-17T08:27:26Z |
    | 942ebcec-4cd1-4386-99a1-7152e7a7b9be | task2_instance | ACTIVE | public=192.168.100.229, 2001:db8::f1                   | Ubuntu-18.04             | ds512M  | 2021-08-15T14:43:10Z |
    | 86256d63-2d36-4aed-a67d-81e6861f12ec | test_instance  | ACTIVE | public=192.168.100.178, 2001:db8::2c7                  | cirros-0.5.2-x86_64-disk | m1.tiny | 2021-08-15T13:10:22Z |
    | ec2c6265-3d0a-4ed2-81c2-7a5f748e9d8f | task1_instance | ACTIVE | public=192.168.100.144, 2001:db8::122                  | cirros-0.5.2-x86_64-disk | m1.tiny | 2021-08-14T03:21:12Z |
    +--------------------------------------+----------------+--------+--------------------------------------------------------+--------------------------+---------+----------------------+

Issue
^^^^^^^^

**재용님이 PR 한 Issue 입니다.**

.. code-block:: python

   # openstackclent/compute/v2/server.py ListServer
    def take_action(self, parsed_args):
        ...
        # 과정 1
            for c in parsed_args.columns:
                ...
                if c in ('Created At', 'created_at'):
                    columns.append('created_at') # Issue
                    column_headers.append('Created At')
        ...
        # 과정 2
        data = compute_client.servers.list(search_opts=search_opts,
                                                   marker=marker_id,
                                                   limit=parsed_args.limit)
        ...
        # 과정 3
        table = (
                column_headers,
                (
                    utils.get_item_properties(
                        s, columns,
                        mixed_case_fields=mixed_case_fields,
                        formatters={
                            'OS-EXT-STS:power_state': PowerStateColumn,
                            'Networks': format_columns.DictListColumn,
                            'Metadata': format_columns.DictColumn,
                        },
                    ) for s in data
                ),
            )
        return table

"과정 2" 수행 시 command 를 실행하게 되면 data 는 \"server list\" 에 해당하는 서버 인스턴스들을 가리키는 객체가 된다.

각 인스턴스들의 \"Created At\" 값이 \"created_at\" 이 아닌 \"created\" 키에 값이 저장되어 있다.

openstack server list -c \"Created At\" 을 수행하게 되면 빈 테이블이 출력된다.

⇒ 이유: "과정 3"에서 columns 튜플에 존재하는 값들과 각 서버 인스턴스들이 가지고 있는 키 값을 비교해 같은 키들만을 테이블에 출력(정확히 말하면 table 구성)한다. columns 에 추가된 값은 \"created_at\" 이고 서버 인스턴스가 가진 키는 \"created\" 이므로 매칭이 안되어 출력이 안되는 것이다.

그래서 "과정 1"의 columns.append('created_at') command를 columns.append(\'created\') 로 수정해야한다.