========================================
openstack server list 에 field 추가하기
========================================

-----
목표
-----
  - openstack server list 의 기본 결과 필드에 "Project ID" 추가하기
  - openstack server list 의 기본 결과 필드에 "Created At" 추가하기

-----

--------------------------------------------------------------------------------
openstack server list 의 기본 결과 필드에 "Project ID" 와 "Created At" 추가하기
--------------------------------------------------------------------------------

앞선 2주차 과제를 통해 ``server list`` 명령어 실행 시 openstackclient/compute/v2/server.py 의 ListServer Class 내 ``take_action()`` 함수에서
서버(인스턴스) 정보를 table 형태로 반환해주는 것을 확인했었다.

이 테이블에 새로운 컬럼을 추가하기 위해 ``take_action()`` 코드를 분석하였고 아래와 같이 column_headers 변수를 초기화하는 부분에
``Project ID`` 와 ``Created At`` 을 추가해주었다.

이 상태에서 openstack 을 실행하면 ``Row has incorrect number of values, (actual) 6!=8 (expected)`` 라는 에러 문구가 출력되면서 CLI 가 정상적으로 동작하지
않는데 이를 통해 column_headers 에 컬럼을 새로 추가하려면 그에 해당하는 데이터 값도 columns 에 추가해주어야 한다는 사실을 알 수 있었다.

.. code-block:: python

    if parsed_args.long:
        ...
    else:
        if parsed_args.no_name_lookup:
            ...
        else:
            columns = (
                'ID',
                'Name',
                'Status',
                'Networks',
                'Image Name',
                'Flavor Name'
            )
        column_headers = (
            'ID',
            'Name',
            'Status',
            'Networks',
            'Image',
            'Flavor',
            'Project ID',
            'Created At'
        )
        mixed_case_fields = []

좀 더 코드를 분석하다보면 아래 부분에서 #support for additional columns 주석을 볼 수 있는데 이 부분에서 힌트를 얻어 Project ID 값을 추가하려면
column_headers(테이블에 컬럼명) 에 ``Project ID`` 를 append 하고 columns(컬럼의 데이터 값) 에는 ``tenant_id`` 를 append 하면 된다는 것을 알 수 있었다.

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

마찬가지로 Created At 컬럼을 추가하려면 column_headers 에 ``Created At`` 을 append 하고 columns 에는 ``created_at`` 을 추가하면 되겠다고 생각하여
``created_at`` 을 append 하였으나 데이터 값이 출력되지 않아서 확인해보니 ``created_at`` 이 아닌 ``created`` 를 append 해주어야 했다.

즉, 위의 코드는 server list 명령어 실행 시 ``take_action(self, parsed_args)`` 함수로 전달되는 인자인 ``parsed_args.columns`` 값이 ``None`` 이기 때문에
실제 수행되지는 않으나 버그로 볼 수 있는데 이에 대해서는 오픈스택 팀의 첫 번째 commit (https://review.opendev.org/c/openstack/python-openstackclient/+/806464)
으로 Gerrit 에 리뷰가 올라가있다.

정리하면 처음 살펴보았던 코드를 아래와 같이 columns 와 column_headers 를 수정하여 아주 간단하게 테이블에 ``Project ID`` 와 ``Created At`` 필드를 추가할 수 있다.

.. code-block:: python

    if parsed_args.long:
        ...
    else:
        if parsed_args.no_name_lookup:
            ...
        else:
            columns = (
                'ID',
                'Name',
                'Status',
                'Networks',
                'Image Name',
                'Flavor Name',
                'tenant_id',
                'created'
            )
        column_headers = (
            'ID',
            'Name',
            'Status',
            'Networks',
            'Image',
            'Flavor',
            'Project ID',
            'Created At'
        )
        mixed_case_fields = []
..

.. image:: ../images/week3/image_1.png
