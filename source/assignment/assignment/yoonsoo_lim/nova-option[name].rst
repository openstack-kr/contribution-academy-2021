[nova] server list --name 옵션 분석
=====================================================================

1. servers.py
---------------------

    * servers.py의 list함수에 search_opts에 옵션과 값을 먼저 전달한다.

    .. code-block:: python

        >>> search_opts
        {'reservation_id': None, 'ip': None, 'ip6': None, 'name': 'cir', ...생략...}

    * ``search_opts`` 변수의 key에 value가 있으면 그것을 qparams에
      저장하고 최종적으론 ``query_string`` 에 저장한다.

    .. code-block:: python

        910  for opt, val in search_opts.items():
        911      # support locked=False from 2.73 microversion
        912      if val or (opt == 'locked' and val is False):
        913          if isinstance(val, str):
        914              val = val.encode('utf-8')
        915          qparams[opt] = val

        ...

        936  if qparams or sort_keys or sort_dirs:
             ...
        947      new_qparams = sorted(items, key=lambda x: x[0])
        948      query_string = "?%s" % parse.urlencode(new_qparams)

    * 924L을 보면 ``/datail`` 이라는것을 ``detail변수`` 에 저장하고

    .. code-block:: python

        922  detail = ""
        923  if detailed:
        924      detail = "/detail"

    * 952L에 보면 novaclient/base.py의 ``_list`` 함수로 값들을 전달한다.

    .. code-block:: python

        952  servers = self._list("/servers%s%s" % (detail, query_string),
        953                       "servers")


---------------------------------------------------------

2. base.py
---------------------

    * ``_list`` 함수에 인자값 url을 확인해보면 아래와 같고
      이것을 ``self.api.client.get`` 에 전달하면 인스턴스명이 ``cir`` 로 시작하는
      인스턴스의 정보들을 반환한다.

    .. code-block:: python

        >>> url
        '/servers/detail?name=cir'

    .. code-block:: python

        252  else:
        253     resp, body = self.api.client.get(url)

    .. code-block:: python

        >>> body
        {'servers': [{'id': '025de69c-0b9e-44e3-9492-874be161a7ee', 'name': 'cirros123' ...생략
        {'id': '0b7e190e-ea7f-4705-ae7a-1933cde13d09', 'name': 'cirros' ...생략




    * nova api는 자체적으로 --name옵션과 정규표현식을 지원, nova help list에서 --name 옵션 존재
    * glance api는 glance help image-list를 했을때 부터 --name이라는 옵션을 지원하지 않음
      하지만 openstack image list 에선 --name이 존재, 하지만 정규표현식은 x
      ?name=cir이런식으로 값을 전달 하긴 하니 직접 코드를 뜯어봐야 알듯,,,,,