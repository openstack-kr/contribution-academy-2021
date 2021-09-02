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
- ListServer.app.client_manager 에 미리 계산된 주소를 담고 있다.
- self.app.clientmanager 객체에 _auth_required 변수가 false이면, 인증이 필요하지 않으면 명령어가 감춰진다. 인증이 True인 상태로 유지되면 인증 또한 유지된다.
- 이같은 일종의 lock이 가능한 이유는 auth_ref가 auth_required가 True로 바뀌는 순간에 인증을 받아와서 self._auth_ref가 인증 정보를 가진 상태가 되기 때문이다.
- auth_required가 True가 되면 get_endpoint_for_service_type 함수는 service catalogy url를 받아오고, endpoint 정보를 받아와 session에 저장해두게 된다.

.. code-block:: python

    # osc-lib/clientmanager.py
    class ClientManager(object):
        """Manages access to API clients, including authentication."""

        # NOTE(dtroyer): Keep around the auth required state of the _current_
        #                command since ClientManager has no visibility to the
        #                command itself; assume auth is not required.
        _auth_required = False


        @property
        def auth_ref(self):
            """Dereference will trigger an auth if it hasn't already"""
            if (not self._auth_required or
                    self._cli_options.config['auth_type'] == 'none'):
                # Forcibly skip auth if we know we do not need it
                return None
            if not self._auth_ref:
                self.setup_auth()
                LOG.debug("Get auth_ref")
                self._auth_ref = self.auth.get_auth_ref(self.session)
            return self._auth_ref

        def get_endpoint_for_service_type(self, service_type, region_name=None,
                                          interface='public'):
            """Return the endpoint URL for the service type."""

            # See if we are using password flow auth, i.e. we have a
            # service catalog to select endpoints from
            if self.auth_ref:
                endpoint = self.auth_ref.service_catalog.url_for(
                    service_type=service_type,
                    region_name=region_name,
                    interface=interface,
                )
            else:
                # Get the passed endpoint directly from the auth plugin
                endpoint = self.auth.get_endpoint(
                    self.session,
                    interface=interface,
                )
            return endpoint



- cmd.auth_required가 True를 client_manager._auth_required에 전달되면 인증된 상태로 전환된다.

.. code-block:: python

    # Cliff/app.py/APP
    def run_subcommand(self, argv):
        cmd_factory, cmd_name, sub_argv = subcommand # Novaclient.Listserver 가 해답 클래스임을 넘겨줌
        ...
        cmd = cmd_factory(self, self.options, **kwargs) # Listserver를 빌딩
        result = 1
        err = None
        try:
            self.prepare_to_run_command(cmd) #

    # osc_lib/shell.py
    def prepare_to_run_command(self, cmd):
        """Set up auth and API versions"""
        ...
        # NOTE(dtroyer): If auth is not required for a command, skip
        #                get_one()'s validation to avoid loading plugins
        validate = cmd.auth_required

        # NOTE(dtroyer): Save the auth required state of the _current_ command
        #                in the ClientManager
        self.client_manager._auth_required = cmd.auth_required # cmd.auth_required가 True를 client_manager._auth_required에 전달되면 인증된 상태로 전환된다.



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