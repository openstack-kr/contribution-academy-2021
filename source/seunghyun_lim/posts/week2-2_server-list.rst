Week 2. openstack server list 명령어 동작 원리 파악 (2부)
====================================================================================

openstack server list 는 환경변수에 등록된 프로젝트의 모든 인스턴스를 출력하는 명령어입니다.

python-openstackcli 의 코드 내부에서 이 명령어를 처리하기 위한 절차를 분석 후 자유롭게 보고서로 작성해주세요.

Index
--------------
3. `openstackcli 는 어떻게 nova api 주소를 알아내나요?`_
4. `nova 의 어떤 API를 호출하여 결과를 받아오나요? ( 어떤 URI 를 호출하나요? )`_
5. `결과를 이쁘게 table 형식으로 출력해주는 함수는 무엇일까요?`_

openstackcli 는 어떻게 nova api 주소를 알아내나요?
---------------------------------------------------------------------------------------------------
- "server list"를 실행하는 python-openstackclient/openstackclient/compute/v2/server.py을 보면, ListServer ListServer.app.client_manager에 미리 계산된 주소를 담고 있다.
- 여기서 ListServer.app은 python-openstackclient/openstackclient/shell.py에서 선언되는 OpenStackShell 클래스이다.
- nova api 주소를 어떻게 알아내는지 알고 싶다면, OpenStackClient 객체의 client_manager가 미리 계산된 주소를 받아오는 과정을 추적해보자.

ClientManager 선언
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
.. code-block:: python

    # osc-lib/osc_lib/shell.py/OpenStackShell
    def initialize_app(self, argv):
        ...
        self.client_manager = clientmanager.ClientManager(
            cli_options=self.cloud,
            api_version=self.api_version,
            pw_func=prompt_for_password,
        )

    # osc-lib/clientmanager.py
    class ClientManager(object):
        """Manages access to API clients, including authentication."""

        _auth_required = False

- **ClientManager는 위의 클래스 설명처럼 nova와 같은 API client의 접근을 가능하게 하는 관리 클래스이다.**
- self.app.clientmanager 객체에 **_auth_required** 변수는 client 정보를 client_manager 객체에 받아올 지 여부를 관리하고 있다.
- prepare_to_run_command를 run 실행 전 호출하면 client_manager는 _auth_required를 True로 받아와 client 정보를 session에 받아 저장하게 된다. 이는 이후 ListServer 클래스


prepare_to_run_command의 활약
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

- prepare_to_run_command에서는 API client 사용 전 auth 설정을 통해 keystone에서 다른 컴포넌트들의 API access 정보를 받아오는 역할을 수행한다.

.. code-block:: python

    # Cliff/app.py/APP
    def run_subcommand(self, argv):
        cmd_factory, cmd_name, sub_argv = subcommand # Novaclient.Listserver 가 해답 클래스임을 넘겨줌
        ...
        cmd = cmd_factory(self, self.options, **kwargs) # Listserver를 빌딩
        try:
            self.prepare_to_run_command(cmd)

    # osc_lib/shell.py
    def prepare_to_run_command(self, cmd):
        """Set up auth and API versions"""

        # ** 인증와 API 정보를 받을 수 있도록 client_manager._auth_required를 True로 만들어줌
        self.client_manager._auth_required = cmd.auth_required # True

        # Validate auth options
        # Push the updated args into ClientManager
        ...
        # Set up auth and APi versions
        if cmd.auth_required:
            # ** setup_auth를 통해 다른 API들의 접근 정보를 가지고 있는 keystone과의 connection을 만듬
            self.client_manager.setup_auth() # auth
            if hasattr(cmd, 'required_scope') and cmd.required_scope:
                # let the command decide whether we need a scoped token
                self.client_manager.validate_scope()
            # Trigger the Identity client to initialize
            # ** 다른 API들의 접근 정보를 받아 session에 전달
            self.client_manager.session.auth.auth_ref = \
                self.client_manager.auth_ref
        return

- 핵심적으로 봐야할 것은 _auth_required가 True로 전환된 이 후 **setup_auth()와 auth_ref** 두 객체들이다.
- 또 하나, 코드로는 확인할 수 없는 핵심 아키텍쳐은 keystone이 API들의 access 정보를 전부 가지고 있는 형태라는 점이다.
- openstackclient가 nova와 같은 컴포넌트들의 API를 사용하기 전 keystone의 인증을 받아 keystone과 connection을 먼저 만들고, keystone에 요청을 보내 API들의 접근 정보를 session에 받아온다.
- 이를 잘 생각하며 setup_auth()와 auth_ref의 동작을 살펴보자.

setup_auth의 동작
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
- prepare_to_run_command 함수에서 실행된 setup_auth 동작은 아래와 같다.

.. code-block:: python

    def setup_auth(self):

        # 선택한 auth 타입을 저장
        self.auth_plugin_name = self._cli_options.config['auth_type']

        # auth valid test
        auth.check_valid_authentication_options(
            self._cli_options,
            self.auth_plugin_name,
        )

        # Horrible hack alert...must handle prompt for null password if
        # password auth is requested.
        if (self.auth_plugin_name.endswith('password') and
                not self._cli_options.auth.get('password')):
            self._cli_options.auth['password'] = self._pw_callback()

        # load keystone plugin and make auth url
        self.auth = self._cli_options.get_auth()

        # session을 받아와 연결
        self.session = self._cli_options.get_session()

        # Connection 객체를 만들어준다. cloud region API 정보를 가지고 오고, vendor_hook에 해당하는 패키지를 entrypoint에서 읽어온다.
        self.sdk_connection = connection.Connection(config=self._cli_options)

        self._auth_setup_completed = True


auth_ref의 동작
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
- auth_required가 일종의 lock이 가능한 이유는 auth_ref가 auth_required가 True로 바뀌는 순간에 인증을 받아와서 self._auth_ref가 인증 정보를 가진 상태가 되기 때문이다.
- auth_required가 True가 되면 get_endpoint_for_service_type 함수는 service catalogy url를 받아오고, endpoint 정보를 받아와 session에 저장해두게 된다.

.. code-block:: python

    # osc-lib/clientmanager.py
    class ClientManager(object):
        """Manages access to API clients, including authentication."""

        # auth_ref을 실행시키는 트리거
        _auth_required = False

        @property
        def auth_ref(self):
            # auth_required가 True인지 auth_type이 존재하는 지 확인
            if (not self._auth_required or
                    self._cli_options.config['auth_type'] == 'none'):
                # Forcibly skip auth if we know we do not need it
                return None

            if not self._auth_ref:
                # 다시 setup_auth 실행
                self.setup_auth()
                # 전에 설정했던 self.auth = keystone auth의 get_auth_ref 함수로 다른 API들의 접곤 정보를 가지고 옴
                self._auth_ref = self.auth.get_auth_ref(self.session)
            return self._auth_ref

- self.auth = keystoneauth1/identity/generic/base.py/BaseGenericPlugin
- get_auth_ref(session)은 어떻게 실행될까

.. code-block:: python

    # keystoneauth1/identity/generic/base.py
    class BaseGenericPlugin(base.BaseIdentityPlugin):
        ...
        def get_auth_ref(self, session, **kwargs):
            if not self._plugin:

                # self._plugin = keystoneauth1/identity/v3/base.py/Auth
                self._plugin = self._do_create_plugin(session)

            return self._plugin.get_auth_ref(session, **kwargs)


- self._plugin = keystoneauth1/identity/v3/base.py/Auth
- Auth.get_auth_ref을 더 자세히 보자.

.. code-block:: python

    # keystoneauth1/identity/v3/base.py
    class Auth(BaseAuth):
        ...
        def get_auth_ref(self, session, **kwargs):
            # requests http 메시지 세팅
            headers = {'Accept': 'application/json'}
            body = {'auth': {'identity': {}}}
            ident = body['auth']['identity']
            rkwargs = {}

            for method in self.auth_methods:
                # 메소드 패스워드 이름과 사용자 설정
                name, auth_data = method.get_auth_data(
                    session, self, headers, request_kwargs=rkwargs)
            ...
            # token_url 설정
            token_url = self.token_url
            ...
            # post를 token_url로 보내서 token을 받아온다.
            resp = session.post(token_url, json=body, headers=headers,
                                authenticated=False, log=False, **rkwargs)
            ...
            # X-Subject-Token에 받아온 token을 넣어서 컴포넌트 API 정보들을 받아온다.
            return access.AccessInfoV3(auth_token=resp.headers['X-Subject-Token'],
                                       body=resp_data)

- session.post를 통해서 token_url로 보내는 request 정보

.. code-block:: python

    method = POST
    token_url = http://{my public}/identity/v3/auth/tokens

- AccessInfoV3를 통해 keystone의 안에 API들, nova, glance 등의 접근 정보를 가지고 있는 catalog에서 값을 가지고 오게 된다.
- keystoneauth1/access/access.py/AccessInfoV3의 @property인 service_catalog은 ServiceCatalog.from_token 메소드를 실행한다.
- service_catalog의 from_token은 token 정보를 X-Subject-Token 헤더에 넣어 catalog 데이터를 가져오게 된다.

.. note::

    keystoen에 대해 더 알고 싶다면 https://naleejang.tistory.com/106

nova 의 어떤 API를 호출하여 결과를 받아오나요? ( 어떤 URI 를 호출하나요? )
---------------------------------------------------------------------------------------------------
- URL : "http://211.37.146.251/compute/v2.1/servers/detail"
- novaclient.API : http://211.37.146.251/compute/v2.1
- request url added : /servers/detail
- keystone 인증이 필요하기 때문에 novaclient.client.SessionClient의 get 메소드를 이용하여 세션에 있는 인증을 사용하여 결과를 받아온다.

.. code-block:: python

    # python_novaclient-17.5.0-py3.9.egg/novaclient/base.py
    class Manager(HookableMixin):

        def _list(self, url, response_key, obj_class=None, body=None, # url : /servers/detail
                  filters=None):
            if filters:
                url = utils.get_url_with_filter(url, filters)
            if body:
                resp, body = self.api.client.post(url, body=body)
            else:
                resp, body = self.api.client.get(url) # url에 get method 호출

    # python_novaclient-17.5.0-py3.9.egg/novaclient/v2/servers.py
    class ServerManager(base.BootingManagerWithFind):
        def list(self, detailed=True, search_opts=None, marker=None, limit=None,
             sort_keys=None, sort_dirs=None):
            if detailed:
                detail = "/detail"

            result = base.ListWithMeta([], None)
            while True:
                ...
                if qparams or sort_keys or sort_dirs:
                    ...
                else:
                    query_string = ""

                servers = self._list("/servers%s%s" % (detail, query_string), # /servers/detail
                                     "servers")
                result.extend(servers)
                result.append_request_ids(servers.request_ids)
            return result


결과를 이쁘게 table 형식으로 출력해주는 함수는 무엇일까요?
---------------------------------------------------------------------------------------------------
- listener가 parser의 내용와 nova api를 완료한 결과를 받아와 Cliff.lister.py/Lister의 produce_output에서 Cliff/formatter/table.py로 전달
- table의 emit_list에서 실행 결과를 pretty_table의 get_string을 이용하여 그려준다.

.. code-block::python

    # Cliff/lister.py/Lister
    def produce_output(self, parsed_args, column_names, data):
        ...
        self.formatter.emit_list(
            columns_to_include, data, self.app.stdout, parsed_args,
        )

    # Cliff/formatter/table.py
    class TableFormatter(base.ListFormatter, base.SingleFormatter):
        def emit_list(self, column_names, data, stdout, parsed_args):
            # table의 형태를 만들어 주고
            x = prettytable.PrettyTable(
                column_names,
                print_empty=parsed_args.print_empty,
            )
            x.padding_width = 1

            # Add rows if data is provided
            if data:
                # rows를 추가해주고
                self.add_rows(x, column_names, data)

            # Choose a reasonable min_width to better handle many columns on a
            # narrow console. The table will overflow the console width in
            # preference to wrapping columns smaller than 8 characters.
            min_width = 8
            self._assign_max_widths(
                stdout, x, int(parsed_args.max_width), min_width,
                parsed_args.fit_width)

            # stdout으로 write 해서 출력 되도록 한다.
            formatted = x.get_string()
            stdout.write(formatted)
            stdout.write('\n')
            return