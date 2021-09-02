OpenStack 팀 2주차 : openstack server list 명령어 동작 원리 파악
================================================================

2주차 2번째 과제에 대한 정리 내용입니다.

1. 인자로 입력받은 server list 를 어떻게 구별해내는가? < 필수 >
2. server list  라는 명령어를 처리하는 파일은 무엇인가? <필수 >
3. openstackcli 는 어떻게 nova api 주소를 알아내나요?
4. nova 의 어떤 API를 호출하여 결과를 받아오나요? ( 어떤 URI 를 호출하나요? )  < 필수 >
5. 결과를 이쁘게 table 형식으로 출력해주는 함수는 무엇일까요?

0. 상속관계
-------------------------------------------------------------
openstackclient  -> osc_lib  -> clif

OpenStackShell -> OpenStackShell ->  App

1. 인자로 입력받은 server list 를 어떻게 구별해내는가? & 2. server list  라는 명령어를 처리하는 파일은 무엇인가?
-------------------------------------------------------------------------------------------------------------------

1. openstackclient/shell.py

.. code-block:: python

   def main(argv=None):
    if argv is None:
        argv = sys.argv[1:]

    return OpenStackShell().run(argv)


a. main 함수의 OpenStackShell().run(argv) 통해 OpenStackShell object 가 생성된다.
b. 해당 객체의 run() 메소드를 실행한다.

argv = ['server', 'list'] 가 run 메소드의 인자로 넘겨진다.

.. code-block:: python

   # openstackclient/shell.py
   # OpneStackShell 객체 선언부
   # 부모 클래스인 osc_lib/shell.OpenStackShell 를 상속한다.
   class OpenStackShell(shell.OpenStackShell):

    def __init__(self):

        super(OpenStackShell, self).__init__(
            description=__doc__.strip(),
            version=openstackclient.__version__,
            command_manager=commandmanager.CommandManager('openstack.cli'), # command manager 를 조금 더 들어가보자.
            deferred_help=True)



.. code-block:: python

   # cliff/commandmanager.py
       def __init__(self, namespace, convert_underscores=True):
        self.commands = {}
        self._legacy = {}
        self.namespace = namespace
        self.convert_underscores = convert_underscores
        self.group_list = []
        self._load_commands()   # load_commands 를 통해

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

이와 같이 객체 선언을 마친후, run command 를 실행한다.
상속관계에 따라 run 은 clif 의 run 메소드가 실행된다.

.. code-block:: python

   # clif/app.py
           try:
            self.options, remainder = self.parser.parse_known_args(argv)    # remainder = ['server', 'list']
            self.configure_logging()
            self.interactive_mode = not remainder
            if self.deferred_help and self.options.deferred_help and remainder:
                self.options.deferred_help = False
                remainder.insert(0, "help")
            self.initialize_app(remainder)

run 실행 중 initialize_app 라는 메소드가 있다.
remainder =  ['server', 'list'] 를 인자로 받는다.

.. code-block:: python

   # openstackclient/shell.py
       def initialize_app(self, argv):
        super(OpenStackShell, self).initialize_app(argv)


부모 클래스인 osc_lib/shell.py 로 이동해보자.

.. code-block:: python

   # osc_lib/shell.py
       def initialize_app(self, argv):
        """Global app init bits:

        * set up API versions
        * validate authentication info
        * authenticate against Identity if requested
        """

        self._load_plugins()

        self._load_commands()

객체를 초기화 하는 내용이다.
API version 에 대한 내용, authentication info 등이 정의가 된다고 한다.
load_plugin 메소드로 들어가보자.

.. code-block:: python

   # openstackclient/shell.py
   def _load_plugins(self):
       ...
       cmd_group = 'openstack.' + api.replace('-', '_') + version  # cmd_group: 'openstack.compute.v2'
       self.command_manager.add_command_group(cmd_group)


cmd_group 이 선언되었다. 이를 더 자세하게 알아보자.


.. code-block:: python

    # cliff/commandmanager.py

        def add_command_group(self, group=None):    # group = 'openstack.compute.v2'
        """Adds another group of command entrypoints"""
        if group:
            self.load_commands(group)


        def load_commands(self, namespace):     # namespace = 'openstack.compute.v2'
        """Load all the commands from an entrypoint"""
        self.group_list.append(namespace)
        for ep in stevedore.ExtensionManager(namespace):
            LOG.debug('found command %r', ep.name)
            cmd_name = (ep.name.replace('_', ' ')
                        if self.convert_underscores
                        else ep.name)
            self.commands[cmd_name] = ep.entry_point
        return

self = <osc_lib.command.commandmanager.CommandManager object at 0x10a292a90>
이와 같고 이와 같은 명령어들이 commands 안에 들어있음을 볼 수 있다.

.. code-block::

    commands = {dict: 37} {'command list': EntryPoint(name='command_list', value='openstackclient.comm ...
     'command list' = {EntryPoint: 3} EntryPoint(name='command_list', value='openstackclient.common.module:ListCommand', group='openstack.cli')
     'module list' = {EntryPoint: 3} EntryPoint(name='module_list', value='openstackclient.common.module:ListModule', group='openstack.cli')
     'help' = {EntryPointWrapper} <cliff.commandmanager.EntryPointWrapper object at 0x10ace4b20>
     'complete' = {EntryPointWrapper} <cliff.commandmanager.EntryPointWrapper object at 0x10ace4970>
     'aggregate add host' = {EntryPoint: 3} EntryPoint(name='aggregate_add_host', value='openstackclient.compute.v2 ...
     ...


정리하자면, clientmanager => commandmanager => stevedore 흐름으로
어딘가에 저장된 정보를 가져오는 구조라고 할 수 있다.

_load_commands() 도 같은 구조이다.


+) stevedore 란 무엇인가
""""""""""""""""""""""""""""""
여러개의 plug-in 을 동적으로 로딩하게 해주는 라이브러리.
애플리케이션을 실행중에 라이브러리를 로딩하고 싶을때 사용.
여기선 openstack 에서 명령어를 사용하기 위해 모듈을 로딩할때 사용.


self.initialize_app(remainder) 을 통해 필요한 모듈,
entrypoint 로 부터 받은 command 들이
key: "server list", value: serverlist 으로 OpenStackShell object 에 업로드 된다.


.. code-block:: python

    # cliff/app.py
       def run(self, argv):
        """Equivalent to the main program for the application.

        :param argv: input arguments and options
        :paramtype argv: list of str
        """
        try:
            self.options, remainder = self.parser.parse_known_args(argv)
            self.configure_logging()
            self.interactive_mode = not remainder
            if self.deferred_help and self.options.deferred_help and remainder:
                self.options.deferred_help = False
                remainder.insert(0, "help")
            self.initialize_app(remainder)  # remainder = ['server', 'list']
            self.print_help_if_requested()

        result = 1
        if self.interactive_mode:
            result = self.interact()
        else:
            try:
                result = self.run_subcommand(remainder)     # remainder = ['server', 'list']
            except KeyboardInterrupt:
                return _SIGINT_EXIT
        return result

결론적으로 파싱했던 remainder 를 통해서 run_subcommand(remainder) 를 호출한다.

.. code-block:: python

   # cliff/app.py
       def run_subcommand(self, argv):
        try:
            subcommand = self.command_manager.find_command(argv)

   # osc_lib/shell.py
   def find_command(self, argv):
            if name in self.commands:
                found = name  # name = 'server list'
            ...
            if found:   # name 을 key 로 ep 를 가져온다.
                cmd_ep = self.commands[found]
                if hasattr(cmd_ep, 'resolve'):
                    cmd_factory = cmd_ep.resolve()
                else:
                    # NOTE(dhellmann): Some fake classes don't take
                    # require as an argument. Yay?
                    arg_spec = inspect.getfullargspec(cmd_ep.load)
                    if 'require' in arg_spec[0]:
                        cmd_factory = cmd_ep.load(require=False)
                    else:
                        cmd_factory = cmd_ep.load()     # stevdore 를 통해 동적으로 class 로딩

여기서 self.command 는 전에 로딩했던 모든 명령어들이 command 변수에 저장되어 있다.
key - value 로 ep 값을 가져온다.

.. code-block::

   'server list' = {EntryPoint: 3} EntryPoint(name='server_list', value='openstackclient.compute.v2.server:ListServer', group='openstack.compute.v2')
     extras = {list: 0} []
     group = {str} 'openstack.compute.v2'
     name = {str} 'server_list'
     pattern = {Pattern} re.compile('(?P<module>[\\w.]+)\\s*(:\\s*(?P<attr>[\\w.]+))?\\s*(?P<extras>\\[.*\\])?\\s*$')
     value = {str} 'openstackclient.compute.v2.server:ListServer'
     0 = {str} 'server_list'
     1 = {str} 'openstackclient.compute.v2.server:ListServer'
     2 = {str} 'openstack.compute.v2'
     __len__ = {int} 3


.. code-block:: python

   # cliff/app.py
   cmd = cmd_factory(self, self.options, **kwargs)  # class 를 cmd 로 인스턴스화 시킨다.
   # cmd = {ListServer} <openstackclient.compute.v2.server.ListServer object at 0x1103ff640>

        try:
            self.prepare_to_run_command(cmd)
            full_name = (cmd_name
                         if self.interactive_mode
                         else ' '.join([self.NAME, cmd_name])
                         )
            cmd_parser = cmd.get_parser(full_name)
            try:
                parsed_args = cmd_parser.parse_args(sub_argv)
            except SystemExit as ex:
                raise cmd2.exceptions.Cmd2ArgparseError from ex
            result = cmd.run(parsed_args)   # run 을 호출한다.




.. code-block:: python

   # osc_lib/command/command.py
   class Command(command.Command, metaclass=CommandMeta):

    def run(self, parsed_args):
        self.log.debug('run(%s)', parsed_args)
        return super(Command, self).run(parsed_args)


.. code-block:: python

   # cliff/display.py
       def run(self, parsed_args):
        parsed_args = self._run_before_hooks(parsed_args)
        self.formatter = self._formatter_plugins[parsed_args.formatter].obj
        column_names, data = self.take_action(parsed_args)      # 수행부
        column_names, data = self._run_after_hooks(parsed_args,
                                                   (column_names, data))
        self.produce_output(parsed_args, column_names, data)
        return 0


self = {List Server} <openstackclient.compute.v2.server.ListServer object at 0x106473370>
즉, 인자로 전달받은 값을 저장해놓은 command list 에서 꺼낸 값이다.
**따라서, List Server 클래스의 take_action 이 수행된다.**

.. note::
    server list 명령어를 처리해주는 파일은 **openstack/python-openstackclient/openstackclient/compute/v2/server.py 이다.**

결론
""""""""""""""""""""""""""""""
=> plug-in 로딩

=> 모듈을 command manager 갖고 있다

=> 인자로 넘어온게 command list 에 있으면 인자에 맞는 EP 를 반환해준다.

=> EP 를 로딩시킨다.

=> 해당 클래스의 take actions 라는 함수를 실행시킨다.



3. openstackcli 는 어떻게 nova api 주소를 알아내나요? & 4. nova 의 어떤 API를 호출하여 결과를 받아오나요?
----------------------------------------------------------------------------------------------------------


server list 명령어를 처리하는 과정중,


.. code-block:: python

   # cliff/display.py
   class DisplayCommandBase(command.Command, metaclass=abc.ABCMeta):
    """Command base class for displaying data about a single object.
    """
    def __init__(self, app, app_args, cmd_name=None):
        super(DisplayCommandBase, self).__init__(app, app_args,
                                                 cmd_name=cmd_name)
        self._formatter_plugins = self._load_formatter_plugins()

    ...

    # list server 의 수행부
    # /compute/v2/server.py 의 take_action 을 수행한다.
    def run(self, parsed_args):     # self = {ListServer}<openstackclient.compute.v2.server.ListServer object at 0x103e9a190>
        parsed_args = self._run_before_hooks(parsed_args)
        self.formatter = self._formatter_plugins[parsed_args.formatter].obj
        column_names, data = self.take_action(parsed_args)
        column_names, data = self._run_after_hooks(parsed_args,
                                                   (column_names, data))
        self.produce_output(parsed_args, column_names, data)
        return 0


server.py 의 take_action 메소드
3 가지의 작업을 수행한다.

.. code-block:: python

       def take_action(self, parsed_args):
        compute_client = self.app.client_manager.compute    # 1
        identity_client = self.app.client_manager.identity  # 2
        image_client = self.app.client_manager.image        # 3

3 개의 메서드 모두 clientmanager.py 로 이동해서
ClientCache 객체를 반환한다.


.. code-block:: python

   # osc_lib/clientmanager.py
   class ClientCache(object):
    """Descriptor class for caching created client handles."""

    def __init__(self, factory):
        self.factory = factory
        self._handle = None

    def __get__(self, instance, owner):
        # Tell the ClientManager to login to keystone
        if self._handle is None:
            try:
                self._handle = self.factory(instance)       # ???? => client.py 로 이동
            except AttributeError as err:
                # Make sure the failure propagates. Otherwise, the plugin just
                # quietly isn't there.
                raise exceptions.PluginAttributeError(err) from err
        return self._handle



.. code-block:: python

   def make_client(instance):
    """Returns a compute service client."""

      compute_api = utils.get_client_class(
        API_NAME,
        version.ver_major,
        COMPUTE_API_VERSIONS,
    )
    ...

와 같은 과정을 3가지 메소드 모두 공통적으로 거친다.

해당 메소드를 실행하고 나면

.. code-block::

   compute_client = {Client} <novaclient.v2.client.Client object at 0x103ec9b50>
   image_client = {Proxy}  <openstack.image.v2._proxy.Proxy object at 0x103eb9910>
   identity_client = {Client} <keystoneclient.v3.client.Client object at 0x103e15c40>

값을 갖는다.


우선 compute client 의 정보를 확인해보자.

.. code-block::

   compute_client = {Client} <novaclient.v2.client.Client object at 0x10302aca0>
     agents = {AgentsManager} <novaclient.v2.agents.AgentsManager object at 0x10302a7f0>
     aggregates = {AggregateManager} <novaclient.v2.aggregates.AggregateManager object at 0x10302ac40>
     api = {APIv2} <openstackclient.api.compute_v2.APIv2 object at 0x10302ae50>
      HEADER_NAME = {str} 'OpenStack-API-Version'
      SERVICE_TYPE = {str} ''
      endpoint = {str} 'http://211.37.148.101/compute/v2.1'

compute client 로 접근하기 위한 endpoint 를 알 수 있다.
**endpoint = {str} 'http://211.37.148.101/compute/v2.1'**


생성한 인스턴스를 이용해 servers.list 메소드를 호출한다.

.. code-block:: python

   # compute/v2/server.py
           data = compute_client.servers.list(search_opts=search_opts,
                                           marker=marker_id,
                                           limit=parsed_args.limit)

드디어, list 메소드는 servers.py 안에 구현되어 있다.

.. code-block:: python


       # novaclient/v2/servers.py
       def list(self, detailed=True, search_opts=None, marker=None, limit=None,
             sort_keys=None, sort_dirs=None):
        """
        Get a list of servers.

        """
        ...

        detail = ""
        if detailed:
            detail = "/detail"

        result = base.ListWithMeta([], None)
        while True:
            ...
            # _list 메소드를 통해 "demo-instance" 를 불러오는 과정
            servers = self._list("/servers%s%s" % (detail, query_string),
                                 "servers")
            result.extend(servers)
            result.append_request_ids(servers.request_ids)
            ...
        return result


self._list("/servers%s%s" % (detail, query_string),"servers") 를 통해 instance 를 불러온다.

url = {str} '/servers/detail'
response_key = {str} 'servers'

.. code-block:: python

   # novaclient/base.py
   def _list(self, url, response_key, obj_class=None, body=None, filters=None):
      if filters:
         url = utils.get_url_with_filter(url, filters)
      if body:
         resp, body = self.api.client.post(url, body=body)
      else:
         resp, body = self.api.client.get(url)      # get 방식으로 호출하는 것을 알 수 있다.



self.api.client.get(url) 는 keystoneauth1/adapter.py 를 호출한다.

.. code-block:: python

   # keystoneauth1/adapter.py
        def get(self, url, **kwargs):
        return self.request(url, 'GET', **kwargs)

resp = 200
body = instance 정보

가 반환된다.



결론
""""""""""""""""""""""""""""""
=> compute_client 객체를 만든다.

=> novaclient/v2/servers.py 의 list 메소드 실행 (url 전달)

=> novaclient/base.py 의 _list 메소드 실행

=> keystoneauth1/adapter.py 에서 get 방식 호출



**http://211.37.148.101/compute/v2.1/servers/detail** 을 통해 호출한다.



5. 결과를 이쁘게 table 형식으로 출력해주는 함수는 무엇일까요?
----------------------------------------------------------------


.. code-block:: python

   # cliff/display.py
       def run(self, parsed_args):
        parsed_args = self._run_before_hooks(parsed_args)
        self.formatter = self._formatter_plugins[parsed_args.formatter].obj
        column_names, data = self.take_action(parsed_args)
        column_names, data = self._run_after_hooks(parsed_args,
                                                   (column_names, data))
        self.produce_output(parsed_args, column_names, data)        # ****
        return 0


self = {ListServer} <openstackclient.compute.v2.server.ListServer object at 0x1053288e0>

.. code-block:: python

   # cliff/lister.py
   def produce_output(self, parsed_args, column_names, data):

        ...
        columns_to_include, selector = self._generate_columns_and_selector(
            parsed_args, column_names,
        )

data = {tuple} ('ID', 'Name', 'Status', 'Networks', 'Image', 'Flavor')
이와 같이 column 값을 확인 할 수 있었다.

parsed_args 와 column_names, data 를 인자로 전달받아
_generate_columns_and_selector 메소드를 호출한다.


.. code-block:: python

   # cliff/display.py
       def _generate_columns_and_selector(self, parsed_args, column_names):
        """Generate included columns and selector according to parsed args.

        :param parsed_args: argparse.Namespace instance with argument values
        :param column_names: sequence of strings containing names
                             of output columns
        """

이 메소드를 통해  ('ID', 'Name', 'Status', 'Networks', 'Image', 'Flavor') 이 반환되어
table 의 column 을 알 수 있다.

반환 받고 다시 lister.py 로 돌아간다.

.. code-block:: python

   # cliff/lister.py
   def produce_output(self, parsed_args, column_names, data):
    ...
        self.formatter.emit_list(
            columns_to_include, data, self.app.stdout, parsed_args,
        )


emit_list 메소드를 통해 cliff/table.py  로 이동한다.

.. code-block:: python

   # cliff/table.py
   class TableFormatter(base.ListFormatter, base.SingleFormatter):
   ...

       def emit_list(self, column_names, data, stdout, parsed_args):
        x = prettytable.PrettyTable(
            column_names,
            print_empty=parsed_args.print_empty,
        )
        x.padding_width = 1



table 을 만들때 PrettyTable 클래스의 인스턴스를 생성한다.

.. code-block:: python

   # prettytable/prettytable.py
   class PrettyTable:
    def __init__(self, field_names=None, **kwargs):
    ...


결론
""""""""""""""""""""""""""""""
=> cliff/display.py 에서 produce_output 메소드 호출

=> cliff/lister.py 에서 _generate_columns_and_selector 호출 : columns_to_include, selector 정의

=> cliff/lister.py 에서 emit_list 호출

=> cliff/table.py 에서 prettytable.PrettyTable 호출




