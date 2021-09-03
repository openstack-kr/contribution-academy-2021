===========================================================
2주차 과제-2 - openstack server list 명령어 동작 원리 파악
===========================================================


.. toctree::
   :maxdepth: 2

contents
---------------
- `1. 인자로 입력받은 server list 를 어떻게 구별해내는가?`_
- `2. server list  라는 명령어를 처리하는 파일은 무엇인가?`_
- `3. openstackcli 는 어떻게 nova api 주소를 알아내나요?`_
- `4. nova 의 어떤 API를 호출하여 결과를 받아오나요? ( 어떤 URI 를 호출하나요? )`_
- `5. 결과를 이쁘게 table 형식으로 출력해주는 함수는 무엇일까요?`_

1. 인자로 입력받은 server list 를 어떻게 구별해내는가?
-------------------------------------------------------------

결론
"""""""""""""
.. note::
    OpenStackShell object 는 주어진 인자(예: server list)를 처리하기 전에 각 API version 에 대한 혹은 cli, common, extension 등과 같은 group 이 추가가 되고 각 group 에 해당하는 command 들이 dict 형식(key: \"server list\", value: serverlist에 대한 EntryPoint object)으로 OpenStackShell object 에 업로드 된다.
    그 후 주어진 인자 값이 해당 객체 내 command dict 에 존재하는 지 확인한다.

도출 과정
"""""""""""""

1.1 API Version 에 대한 cmd group 의 command 들 업로드
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

1. openstackclient/shell.py

main 함수의 OpenStackShell().run(argv) 통해 OpenStackShell object 가 생성되고 해당 객체의 run() 메소드를 실행한다. run 메소드 인자로 argv = [\"server\", \"list\"] 가 넘겨진다.

2. osc_lib/shell.py

openstackclient/shell.py OpenstackShell 클래스는 osc_lib/shell.py 의 OpenStackShell 클래스를 상속 받는다.

.. code-block:: python

   def run(self, argv):
        ret_val = 1
        self.command_options = argv
        try:
            print("A")
            # super().run(argv) => osc_lib/shell.py/OpenStackShell 은 App 을 상속 받으므로 App 클래스의 run 메소드를 실행한다.
            ret_val = super(OpenStackShell, self).run(argv)
            print("B")
            return ret_val
        except ...

를 수행한 결과가 아래와 같다는 것을 알 수 있었다.

.. code-block:: bash

    A
    server list 의 결과값 (table)
    B

ret_val = **super(OpenStackShell, self).run(argv) 을 통해 결과값이 출력** 된다는 것을 알 수 있다.

3. venv/lib/python3.9/site-packages/cliff/app.py/App

.. code-block:: python

    def run(self, argv):
        try:
            self.options, remainder = self.parser.parse_known_args(argv) # remainder = ["server", "list"]
            self.configure_logging()
            self.interactive_mode = not remainder
            if self.deferred_help and self.options.deferred_help and remainder:
                self.options.deferred_help = False
                remainder.insert(0, "help")
            self.initialize_app(remainder) # OpenStackShell object API version을 설정하고 인증 정보를 확인한다.
            self.print_help_if_requested()
            ...

4. openstackclient/shell.py/OpenStackShell

initialize_app 메소드   super(OpenStackShell, self).initialize_app(argv) 을 통해 osc_lib/shell.py/OpenStackShell initialize_app 메소드를 호출한다.

5. stevedore 를 통해 plugin 들을 load 해주자!

.. code-block:: python

    # osc_lib/shell.py/OpenStackShell
    def initialize_app(self, argv):
        ...
        self._load_plugins()

        self._load_commands()
        ...

5.1 self._load_plugins() 실행

.. code-block:: python

    # openstackclient/shell.py/OpenStackShell
    def _load_plugins(self):
		for mod in clientmanager.PLUGIN_MODULES:
		...
		if version_opt:
			api = mod.API_NAME # "compute"
		...
		version = '.v' + version_opt.replace('.', '_').split('_')[0]
		cmd_group = 'openstack.' + api.replace('-', '_') + version
		self.command_manager.add_command_group(cmd_group)
		...

- PLUGIN_MODULES 는 다음과 같다.

.. code-block:: bash

    [openstack.cli.base]
    compute = openstackclient.compute.client
    identity = openstackclient.identity.client
    image = openstackclient.image.client
    network = openstackclient.network.client
    object_store = openstackclient.object.client
    volume = openstackclient.volume.client

각 PLUGIN_MODULES 에 해당하는 command_group 과 command 들을 CommandManager obejct 에 업로드한다.

5.1.1 venv/lib/python3.9/site-packages/cliff/commandmanager.py/CommandManager

.. code-block:: python

    def add_command_group(self, group=None):
        """Adds another group of command entrypoints"""
        if group:
            self.load_commands(group)

- self : commandmanager.CommandManager object
- group : \"openstack.compute.v2\"
- load_commands(group) 호출

.. code-block:: python

    def load_commands(self, namespace):
        """Load all the commands from an entrypoint"""
        self.group_list.append(namespace)
        for ep in stevedore.ExtensionManager(namespace):
            LOG.debug('found command %r', ep.name)
            cmd_name = (ep.name.replace('_', ' ')
                        if self.convert_underscores
                        else ep.name)
            self.commands[cmd_name] = ep.entry_point
        return

- **self.group_list.append(namespace)** : command.Command object group_list 에 \"openstack.compute.v2\" 를 추가
- stevedore.ExtensionManager(namespace) 호출 시 namespace 에 해당하는 Extension object 들을 생성 후 반환한다.
    - Extension(ep.name, ep, plugin, obj)
        - **ep.name**: server_list
        - **ep**: EntryPoint 인스턴스
        - **plugin**: ListServer class
        - **obj**: None
- ep: namespace 에 해당하는 Extension 인스턴스들
- ep.name : 예) \"server_list\"
- cmd_name: 예) \"server list\"
- self.commands[cmd_name] = ep.entry_point
    - self: CommandManager object
    - ep.entry_point: EntryPoint 인스턴스

5번의 과정을 통해 각 PLUGIN_MODULES 에 해당하는 cmd_group 과 command 가 CommandManager object 에 업로드 된다.

예를 들어 키 값에 명령어 name 이 들어가고 키 값의 value 값은 EntryPoint 객체가 할당된다.
    - key: \"server list\"
    - value: EntryPoint(name=\'server_list\', value=\'openstackclient.compute.v2.server:ListServer\', group=\'openstack.compute.v2\')

5번의 self._load_plugins() 수행 후

- api_version
    - {\'compute\': \'2.1\', \'identity\': \'3\', \'image\': \'2\', \'network\': \'2\', \'object_store\': \'1\', \'volume\': \'3\'}
- command_manager.CommandManager object
    - group_list
        - [\'openstack.cli\', \'openstack.compute.v2\', \'openstack.identity.v3\', \'openstack.image.v2\', \'openstack.network.v2\', \'openstack.object_store.v1\', \'openstack.volume.v3\']

과정 5 을 통해 openstack.cli 를 제외한 다른 group list 가 추가되고 각 group에 해당하는 command 들이 OpenStackShell 객체의 commands dict에 할당이 되었다.

self._load_commands() 를 통해 group_list 에 \"openstack.common\", \"openstack.extension\" 이 추가가 되었고 그에 해당하는 command 들이 추가되었다.

3~5 번 과정을 통해 각 PLUGIN_MODULES 에 해당하는 cmd_group 과 command 가 CommandManager object 에 업로드 되었다.

1.2 인자 값 server list 가 OpenStackShell object command dict 에 존재 여부 확인
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

인자로 준 값이 위에서 업로드한 OpenStackShell object command dict 에 존재하는지 판별해야한다.

.. code-block:: python

    # venv/lib/python3.9/site-packages/cliff/app.py/App
    class App(object):
		...
		def run(self, argv):
				...
				result = 1
        if self.interactive_mode:
            result = self.interact()
        else:
            try:
                result = self.run_subcommand(remainder) # remainer = ["server", "list"]
            except KeyboardInterrupt:
                return _SIGINT_EXIT
        return result

result = **self.run_subcommand(remainder)** 부분에서 server list 인자가 구별된다.

.. code-block:: python

    # osc_lib/shell.py/OpenStackShell
    class OpenStackShell(app.App):
		...
		def run_subcommand(self, argv):
        self.init_profile()
        try:
            ret_value = super(OpenStackShell, self).run_subcommand(argv)
        finally:
            self.close_profile()
        return ret_value
		...

.. code-block:: python

    # venv/lib/python3.9/site-packages/cliff/app.py/App
    class App(object):
		def run_subcommand(self, argv):
		        try:
		            subcommand = self.command_manager.find_command(argv)
						except ValueError as err:
            # If there was no exact match, try to find a fuzzy match
            the_cmd = argv[0]
            fuzzy_matches = self.get_fuzzy_matches(the_cmd)
            if fuzzy_matches:
                article = 'a'
                if self.NAME[0] in 'aeiou':
                    article = 'an'
                self.stdout.write('%s: \'%s\' is not %s %s command. '
                                  'See \'%s --help\'.\n'
                                  % (self.NAME, ' '.join(argv), article,
                                      self.NAME, self.NAME))
                self.stdout.write('Did you mean one of these?\n')
                for match in fuzzy_matches:
                    self.stdout.write('  %s\n' % match)
            else:
                if self.options.debug:
                    raise
                else:
                    self.LOG.error(err)
            return 2

**find_command(argv) : 이 App 클래스 find_command 메소드를 통해 주어진 server list 인자를 통해 현재 OpenStackShell 에 포함된 command 에 있는 지 확인(구별)한다. 만약 인자로 주어진 명령어가 object 내의 command 에 존재하지 않으면 비슷한 (match 되는) 명령어 리스트를 보여주거나 에러를 발생시킨다.**


2. server list 라는 명령어를 처리하는 파일은 무엇인가?
----------------------------------------------------------------

결론
"""""""""""""

.. note::
    server list 명령어를 처리해주는 파일은 **openstack/python-openstackclient/openstackclient/compute/v2/server.py 이다.**

도출과정
"""""""""""""

.. code-block:: python

    # venv/lib/python3.9/site-packages/cliff/app.py/App
    class App(object):
        def run_subcommand(self, argv):
            try:
                subcommand = self.command_manager.find_command(argv)
                    except ...

                    ...
                    # cmd_factory: <class 'openstackclient.compute.v2.server.ListServer> cmd_name: 'server list' sub_argv: []
                    cmd_factory, cmd_name, sub_argv = subcommand
            kwargs = {}
            if 'cmd_name' in inspect.getfullargspec(cmd_factory.__init__).args:
                kwargs['cmd_name'] = cmd_name # kwargs: {'cmd_name': 'server list'}
            cmd = cmd_factory(self, self.options, **kwargs) # OpenstackShell 객체의 options, server list를 인자로 ListServer object 를 생성해서 cmd에 할당
            result = 1
            err = None
            try:
                self.prepare_to_run_command(cmd)
                full_name = (cmd_name
                             if self.interactive_mode
                             else ' '.join([self.NAME, cmd_name])
                             ) # self.NAME="shell"
                            # full_name: 'shell server list'
                cmd_parser = cmd.get_parser(full_name)
                try:
                    parsed_args = cmd_parser.parse_args(sub_argv)
                except SystemExit as ex:
                    raise cmd2.exceptions.Cmd2ArgparseError from ex
                result = cmd.run(parsed_args)

                ...

- **cmd.get_parser(full_name)**

    해당 명령어를 통해 "shell server list" 을 인자로 넘겨주며openstackclient/compute/v2/server.py/ListServer 클래스의 get_parser 메소드를 호출한다.

- **cmd.run(parsed_args)**

    cmd 는 OpenstackShell 객체의 options, server list를 인자로 생성된 ListServer object 이다.
    run 메소드 실행을 하게되면 osc_lib/command/command.py/Command 클래스의 run 메소드가 실행된다.
    => class ListServer 는 command.Lister 를 상속 받고 command.Lister는 Command 클래스를 상속 받기 때문

.. code-block:: python

   # osc_lib/command/command.py/Command
   class Command(command.Command, metaclass=CommandMeta):
		def run(self, parsed_args):
        self.log.debug('run(%s)', parsed_args)
        return super(Command, self).run(parsed_args)

super(Command, self).run(parsed_args) 실행 시 cliff/display.py/DisplayCommandBase 클래스의 run 메소드를 호출한다.

.. code-block:: python

   # venv/lib/python3.9/site-packages/cliff/display.py
   class DisplayCommandBase(command.Command, metaclass=abc.ABCMeta):
        def run(self, parsed_args):
            parsed_args = self._run_before_hooks(parsed_args)
            self.formatter = self._formatter_plugins[parsed_args.formatter].obj
            column_names, data = self.take_action(parsed_args)
            column_names, data = self._run_after_hooks(parsed_args,
                                                       (column_names, data))
            self.produce_output(parsed_args, column_names, data)
            return 0

- self : <openstackclient.compute.v2.server.ListServer object>
- parsed_args
    - Namespace(formatter='table', columns=[], quote_mode='nonnumeric', noindent=False, max_width=0, fit_width=False, print_empty=False, sort_columns=[], sort_direction=None, reservation_id=None, ip=None, ip6=None, name=None, instance_name=None, status=None, flavor=None, image=None, host=None, all_projects=False, project=None, project_domain=None, user=None, user_domain=None, deleted=False, availability_zone=None, key_name=None, has_config_drive=None, progress=None, vm_state=None, task_state=None, power_state=None, long=False, no_name_lookup=False, name_lookup_one_by_one=False, marker=None, limit=None, changes_before=None, changes_since=None, locked=False, unlocked=False, tags=[], not_tags=[])

openstackclient.compute.v2.server.ListServer 객체의 메소드 take_action 을 수행하며 column_names, data 를 반환 받는다.

**즉, server list 를 처리해주는 파일은 openstack/python-openstackclient/openstackclient/compute/v2/server.py 이다.**


3. openstackcli 는 어떻게 nova api 주소를 알아내나요?
----------------------------------------------------------------

\"server list\" 기준

.. code-block:: python

    # site-packages/cliff/app.py
    class App(object):
        ...

        def run_subcommand(self, argv):
        ...
        self.prepare_to_run_command(cmd)
        ...

- self: OpenStackShell object
- cmd: <openstackclient.compute.v2.server.ListServer object>

.. code-block:: python

    # osc_lib/shell.py
    def prepare_to_run_command(self, cmd):
        ...
        if cmd.auth_required:
            # Trigger the Identity client to initialize
            self.client_manager.session.auth.auth_ref = \

- self: OpenStackShell object

.. code-block:: python

    # osc_lib/clientmanager.py
    @property
    def auth_ref(self): # self: openstackclient.common.clientmanager.ClientManger object
        ...
        if not self._auth_ref:
            ...
            self._auth_ref = self.auth.get_auth_ref(self.session)
        return self._auth_ref

.. code-block:: python

    # site-packages/keystoneauth1/identity/generic/base.py
    def get_auth_ref(self, session, **kwargs):
        if not self._plugin:
            self._plugin = self._do_create_plugin(session)

        return self._plugin.get_auth_ref(session, **kwargs)

- self: <keystoneauth1.identity.generic.password.Password object at 0x7f9068498fa0>
- session: <keystoneauth1.session.Session object at 0x7f9088a7eb20>
- self._plugin: <keystoneauth1.identity.v3.password.Password object at 0x7f9078b7df10>

.. code-block:: python

    # site-packages/keystoneauth1/identity/v3/base.py
    def get_auth_ref(self, session, **kwargs):
        ...
        resp = session.post(token_url, json=body, headers=headers,
                                authenticated=False, log=False, **rkwargs)
        ...

- token_url: \'http://211.37.148.128/identity/v3/auth/tokens\'
- body: \'auth\' 정보
- headers: {\'Accept\': \'application/json\'}

.. code-block:: python

    # site-packages/keystoneauth1/session.py
    def post(self, url, **kwargs):
        """Perform a POST request.

        This calls :py:meth:`.request()` with ``method`` set to ``POST``.

        """
        return self.request(url, 'POST', **kwargs)

.. code-block:: python

    # site-packages/keystoneauth1/session.py
    def request(self, url, method, json=None, original_ip=None,
                user_agent=None, redirect=None, authenticated=None,
                endpoint_filter=None, auth=None, requests_auth=None,
                raise_exc=True, allow_reauth=True, log=True,
                endpoint_override=None, connect_retries=None, logger=None,
                allow=None, client_name=None, client_version=None,
                microversion=None, microversion_service_type=None,
                status_code_retries=0, retriable_status_codes=None,
                rate_semaphore=None, global_request_id=None,
                connect_retry_delay=None, status_code_retry_delay=None,
                **kwargs):

        ...
        resp = send(**kwargs)
        ...
        return resp

resp 는 \*\*kwargs(headers, auth 정보) 를 \'http://211.37.148.128/identity/v3/auth/tokens\' 에서 다음과 같은 \"catalog\" 값으로 nova, keystone, cinder, glance 등 모든 컴포넌트들의 api 주소를 가져온다.

.. code-block:: bash

    # resp 중 "catalog" nova 정보
    "catalog":
    [ ... {"endpoints": [{"id": "e7507720bc274e56b420466613be3f07", "interface": "public", "region_id": "RegionOne", "url": "http://211.37.148.128/compute/v2.1", "region": "RegionOne"}], "id": "3e7dec3e86ea4652ad633484b07fa368", "type": "compute", "name": "nova"}, ... ]

업로드된 정보들은 clientmanger.ClientManager obect 의 session.Session object(session) 에 keystoneauth1.identity.generic.password.Password object(auth)에 keystoneauth1.access.access.AccessInfoV3 object(auth_ref)에  keystoneauth1.access.service_catalog.ServiceCatalogV3 object (\"service_catalog\" )에 저장되어 있다.

4. nova 의 어떤 API를 호출하여 결과를 받아오나요? ( 어떤 URI 를 호출하나요? )
----------------------------------------------------------------------------------
- \'http://211.37.148.128/compute/v2.1\' 를 호출해서 결과 값을 받아온 거 같습니다.

5. 결과를 이쁘게 table 형식으로 출력해주는 함수는 무엇일까요?
----------------------------------------------------------------------------------

결론
"""""""""""""
.. note::
    site-packages/cliff/formatters/tables.py TableFormatter 클래스의 **emit_list 메소드** 에서 결과를 이쁘게 table 형식으로 출력해준다.

도출 과정
"""""""""""""

위에서 server list 인자의 openstackclient/compute/v2/server.py 파일에서의 처리 결과로 column_names 와 data 값을 할당 받았다.

- column_names=('ID', 'Name', 'Status', 'Networks', 'Image', 'Flavor')
- data = 해당 명령어 맞는 값들이 들어있는 generator 이다.

.. code-block:: python

   # venv/lib/python3.9/site-packages/cliff/display.py
   class DisplayCommandBase(command.Command, metaclass=abc.ABCMeta):
       def run(self, parsed_args):
            parsed_args = self._run_before_hooks(parsed_args)
            self.formatter = self._formatter_plugins[parsed_args.formatter].obj
            column_names, data = self.take_action(parsed_args)
            column_names, data = self._run_after_hooks(parsed_args,
                                                       (column_names, data))
            self.produce_output(parsed_args, column_names, data)
            return 0

위에서 도출된 값들을 self.produce_output(parsed_args, column_names, data) 를 수행하며 결과값들이 출력이된다. 여기서 self 는 server list 인자 기준 ListServer object 를 가리킨다.

.. code-block:: python

   class Lister(display.DisplayCommandBase, metaclass=abc.ABCMeta):
		...
		def produce_output(self, parsed_args, column_names, data):
            ...
            self.formatter.emit_list(
                columns_to_include, data, self.app.stdout, parsed_args,
            )
            return 0

self.formatter.emit_list(columns_to_include, data, self.app.stdout, parsed_args,) 를 수행 시 cliff/formatters/table.py/TableFormatter 에 emit_list 메소드가 호출된다.

.. code-block:: python

   def emit_list(self, column_names, data, stdout, parsed_args):

		# column_names로 PrettyTable 객체를 생성해 x에 할당
        x = prettytable.PrettyTable(
            column_names,
            print_empty=parsed_args.print_empty,
        )
        x.padding_width = 1 # 이 값을 변경해보면 table 형식이 달라진다는 것을 알 수 있다.

        if data:
            self.add_rows(x, column_names, data) # 데이터들이 table의 각 행에 입력된다.

        min_width = 8
        self._assign_max_widths(
            stdout, x, int(parsed_args.max_width), min_width,
            parsed_args.fit_width)

        formatted = x.get_string()
        stdout.write(formatted) # 결과값(테이블)을 출력해준다.
        stdout.write('\n')
        return

해당 메소드에서 column_names 와 data 를 테이블 형식으로 만들어 출력까지 수행한다.
