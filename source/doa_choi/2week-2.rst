=========================================================
2주차 과제 - openstack server list 명령어 동작 원리
=========================================================

00. openstack server list 명령어 분석
------------------------------------------

01. 인자로 입력받은 server list 를 구별하는 방법
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

**01-01. openstackclient\\shell.py 메인 함수 내 OpenStackShell 클래스에 정의된 run() 메서드 분석**


시작점인 main 함수 내에서 OpenStakShell 클래스의 인스턴스를 생성하고, 해당 인스턴스의 run() 메서드를 호출합니다.
해당 메서드에 전달되는 인자값은 터미널에 입력한 명령어 문자열인 server list 가 전달됩니다.

 .. code-block:: python

    def main(argv=None):
    if argv is None:
        argv = sys.argv[1:]

    return OpenStackShell().run(argv)


|

**01-02. osc_lib\\shell.py 에서 참조하고 있는 app.py 소스에서 선언된 App 클래스를 상속받고 있는 OpenStackShell 클래스의 run() 메서드 분석**


메인 함수에서 호출한 run() 메서드는 osc_lib\\app.App 클래스를 상속받고, App 클래스의 run 메서드의 기능을 오버라이딩한 메서드로 확인됩니다.
해당 함수에는 상위 클래스의 run() 메서드를 호출하여 반환되는 값을 ret_val 변수에 받도록 되어 있습니다.
상위 클래스의 기능을 사용하므로, 상속 받은 클래스의 run() 메서드 내부로 진입하여 해당 기능을 확인하였습니다.

 .. code-block:: python

    class OpenStackShell(app.App):
        def run(self, argv):
            ret_val = 1
            self.command_options = argv
            try:
                ret_val = super(OpenStackShell, self).run(argv)
                return ret_val

                Exception 발생시 수행할 문장..

                finally:
                self.log.info("END return value: %s", ret_val)


|

**01-03. osc_lib\\app.py 소스의 App 클래스에 정의된 run() 메서드**


App 클래스에 정의된 run() 메서드는 인터랙티브 모드라면, interact() 메서드를 호출하고 아니라면, run_subcommand() 메서드를 호출합니다.
테스트를 진행한 환경은 인터랙티브 모드가 아니므로, result 변수에 전달되는 값은 run_subcommand() 메서드의 반환값입니다.
run_subcommand() 메서드에 전달되는 인자는 remainder 리스트의 데이터입니다. 이 값은 server, list 으로 확인됩니다.

 .. code-block:: python

    class App(object):
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
                self.initialize_app(remainder)
                self.print_help_if_requested()

                Exception 발생시 처리할 구문 생략

            if self.interactive_mode:
                result = self.interact()
            else:
                try:
                    result = self.run_subcommand(remainder)
                except KeyboardInterrupt:
                    return _SIGINT_EXIT
            return result


|

**01-04. osc_lib\\app.py 소스에 선언된 App 클래스에 정의된 run_subcommand() 메서드**


run_subcommand() 메서드에 진입하여 해당 함수를 확인하였습니다. 해당 함수에서는 help 등의 다양한 명령어 정보를 출력합니다.
터미널에서 전달한 값은 cmd.run() 메서드를 호출하면서 명령어에 대한 결과가 출력됩니다. 만일, 사용자가 존재하지 않는 명령어를 입력하였을 경우에는
fuzzy_matches 조건식에 의해 --help 와 관련된 명령어 리스트 목록이 출력되는 것을 확인해볼 수 있었습니다.
가장 먼저 해당 메서드의 첫 행에서 호출되는 find_command() 메서드를 확인하였습니다.

 .. code-block:: python

    class App(object):
        def run_subcommand(self, argv):
            try:
                subcommand = self.command_manager.find_command(argv)
                # print (subcommand)
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
            cmd_factory, cmd_name, sub_argv = subcommand
            kwargs = {}
            if 'cmd_name' in inspect.getfullargspec(cmd_factory.__init__).args:
                kwargs['cmd_name'] = cmd_name
                print(kwargs['cmd_name'])
            cmd = cmd_factory(self, self.options, **kwargs)
            result = 1
            err = None
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
                result = cmd.run(parsed_args)

                Exception 발생시 실행되는 구문 및 finally 생략

            return result


|


**01-05. osc_lib\\commandmanager.py 의 CommandManager 클래스에 정의된 find_command() 메서드**


해당 함수에서는 오픈스택 명령어 중에서 전달받은 인자에 대한 엔트리 포인트 정보를 불러옵니다. 명령어는 found 변수에 담겨 commands = found 형태로 값을 초기화하게 되는데,
commands 를 따라 들어가서 CommandManager 클래스의 생성자를 확인해볼 수 있었습니다. 해당 클래스 내부 load_commands() 메서드를 호출하면서
엔트리 포인트로 기록된 명령어 정보를 로드하여 server list 에 대한 정보를 cmd_ep 에 대입하고 cmd_factory, return_name, search_args 각각의 변수에는
명령어 정보와 server list 명령어에 대한 값이 할당하여 이를 반환합니다.


 .. code-block:: python

    def find_command(self, argv):
        """Given an argument list, find a command and
        return the processor and any remaining arguments.
        """
        start = self._get_last_possible_command_index(argv)
        for i in range(start, 0, -1):
            name = ' '.join(argv[:i])
            search_args = argv[i:] 
            return_name = name
            if name in self._legacy:
                name = self._legacy[name]

            found = None
            if name in self.commands:
                found = name
            else:
                candidates = _get_commands_by_partial_name(
                    argv[:i], self.commands)
                if len(candidates) == 1:
                    found = candidates[0]
            if found:
                cmd_ep = self.commands[found]
                if hasattr(cmd_ep, 'resolve'):
                    cmd_factory = cmd_ep.resolve()
                else:
                    arg_spec = inspect.getfullargspec(cmd_ep.load)
                    if 'require' in arg_spec[0]:
                        cmd_factory = cmd_ep.load(require=False)
                    else:
                        cmd_factory = cmd_ep.load()
                return (cmd_factory, return_name, search_args)
        else:
            raise ValueError('Unknown command %r' %
                             (argv,))


 .. code-block:: python

    class CommandManager(object):
        def __init__(self, namespace, convert_underscores=True):
            self.commands = {}
            self._legacy = {}
            self.namespace = namespace
            self.convert_underscores = convert_underscores
            self.group_list = []
            self._load_commands()

        def _load_commands(self):
            # NOTE(jamielennox): kept for compatibility.
            print ("Load Command test")
            if self.namespace:
                print (self.namespace)
                self.load_commands(self.namespace)

        def load_commands(self, namespace):
            self.group_list.append(namespace)
            for ep in stevedore.ExtensionManager(namespace):
                LOG.debug('found command %r', ep.name)
                cmd_name = (ep.name.replace('_', ' ')
                            if self.convert_underscores
                            else ep.name)
                self.commands[cmd_name] = ep.entry_point
            return

|

**01-06. osc_lib\\app.py 의 App 클래스에 정의된 run_subcommand() 메서드로 돌아와서**


반환되는 값을 확인했으므로, 다시 run_subcommand() 메서드를 확인하였습니다. cmd 변수에 할당된 명령어 정보로 openstackclient\\server.py 소스의 ListServer 클래스에 정의된
get_parser() 메서드로 shell server list 명령어를 전달합니다. get.parser 에서 반환되는 값은 Namespace 정보인 것 같습니다.
다음으로 명령어가 출력되는 지점인 run() 메서드를 확인하였습니다.

 .. code-block:: python

    Namespace(all_projects=False, availability_zone=None, changes_before=None, changes_since=None, columns=[], deleted=False, fit_width=False, flavor=None, formatter='table', has_config_drive=None, host=None, image=None, instance_name=None, ip=None, ip6=None, key_name=None, limit=None, locked=False, long=False, marker=None, max_width=0, name=None, name_lookup_one_by_one=False, no_name_lookup=False, noindent=False, not_tags=[], power_state=None, print_empty=False, progress=None, project=None, project_domain=None, quote_mode='nonnumeric', reservation_id=None, sort_columns=[], sort_direction=None, status=None, tags=[], task_state=None, test=None, unlocked=False, user=None, user_domain=None, vm_state=None) []




|


**01-07. osc_lib\\command.py 의 Command 클래스에 정의된 run() 메서드**


Command 클래스는 command.Command 클래스를 상속받았습니다. run() 메서드에서 반환되는 값은 상위 클래스에 정의된 run() 에서 반환되는 값인 것 같습니다.


 .. code-block:: python

    class Command(command.Command, metaclass=CommandMeta):

        def run(self, parsed_args):
            self.log.debug('run(%s)', parsed_args)
            return super(Command, self).run(parsed_args)


|


**01-08. osc_lib\\command.py 의 Command 추상 클래스와 추상 메서드**


해당 클래스에 정의된 run() 메서드의 주석에서 전달하는 내용과 같이 해당 클래스는 추상 클래스로 확인됩니다.
따라서 이 메서드를 구현한 자식 클래스를 찾습니다.

 .. code-block:: python

    class DisplayCommandBase(command.Command, metaclass=abc.ABCMeta):

        def run(self, parsed_args):
            parsed_args = self._run_before_hooks(parsed_args)
            self.formatter = self._formatter_plugins[parsed_args.formatter].obj
            column_names, data = self.take_action(parsed_args)
            column_names, data = self._run_after_hooks(parsed_args,
                                                    (column_names, data))
            self.produce_output(parsed_args, column_names, data)
            return 0

|


**01-09. python-openstackclient\\server.py 의 ListServer 클래스에 정의된 take_action() 메서드**

이와 같은 과정을 통해서 최종적으로 명령어가 출력되는 지점은 **ListServer 클래스** 인 것 같습니다.


 .. code-block:: python

    class ListServer(command.Lister): # ListServer class inferits command class.
        _description = _("List servers")
        
        def take_action(self, parsed_args):
            compute_client = self.app.client_manager.compute
            identity_client = self.app.client_manager.identity
            image_client = self.app.client_manager.image

            print("Here it is!")

            print(compute_client, identity_client, image_client)
            너무 길어서 생략

|

 .. image:: images/2week_01_server_list.png


|


02.  server list 명령어를 처리하는 파일
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

1번 항목에서 분석한 내용을 통해 해당 명령어를 처리하는 파일은 server.py 파일입니다.

+-----------------------------------------------------------------------+
| **server list 명령어를 처리하는 파일**                                |
+=======================================================================+
| openstack/python-openstackclient/openstackclient/compute/v2/server.py |
+-----------------------------------------------------------------------+

|


1.  openstackcli가 nova api 주소를 알아내는 방법과 명령어의 결과를 받아오기 위해 호출되는 api
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

initialize_app() 메서드에서 API 인증을 진행하고, openstackclient\\compute\\v2\\server.py 의 첫번째 행인 compute_cline = self.app.client_manager.compute => client.make_client()에 의해 클라이언트를 생성하고,
osc_lib\\novaclient\\servers.py 의 ServerManager 클래스의 list() 메서드에서 API 정보를 확인할 수 있었습니다.
즉, **nova의 API는 /compute/servers/detail 로 확인됩니다.**


.. code-block:: python

    class ListServer(command.Lister): # ListServer class inferits command class.
        _description = _("List servers")

        def take_action(self, parsed_args):
            compute_client = self.app.client_manager.compute


|

**openstackclient\\compute\\client.py**

.. code-block:: python

    def make_client(instance):
        """Returns a compute service client."""

        client = nova_client.Client(
        version,
        session=instance.session,
        extensions=extensions,
        http_log_debug=http_log_debug,
        timings=instance.timing,
        region_name=instance.region_name,
        **kwargs
        )

        client.api = compute_api(
        session=instance.session,
        service_type=COMPUTE_API_TYPE,
        endpoint=instance.get_endpoint_for_service_type(
            COMPUTE_API_TYPE,
            region_name=instance.region_name,
            interface=instance.interface,
        )
        

|


**osc_lib\\novaclient\\v2\\client.py**

.. code-block:: python

    class Client(object):
        """Top-level object to access the OpenStack Compute API.

        .. warning:: All scripts and projects should not initialize this class
        directly. It should be done via `novaclient.client.Client` interface.
        """

        self.servers = servers.ServerManager(self)


|


**osc-lib\\novaclient\\v2\\servers.py**

.. code-block:: python

    class ServerManager(base.BootingManagerWithFind):
    resource_class = Server

        def list(self, detailed=True, search_opts=None, marker=None, limit=None, sort_keys=None, sort_dirs=None):

            detail = ""
                if detailed:
                    detail = "/detail"


|

 .. image:: images/2week_03_api.png


 |


04. 명령어의 결과를 예쁘게 table 형식으로 출력해주는 함수
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
osc_lib\\display.py 의 DisplayCommandBase 클래스에 정의된 run() 메서드의 produce_output 에서 
TableFormatter 클래스의 emit_list() 메서드를 호출합니다. 해당 메서드가 명령어의 결과를 출력해주는 함수인 것 같습니다. 


 .. code-block:: python
 
    class DisplayCommandBase(command.Command, metaclass=abc.ABCMeta):

        def run(self, parsed_args):
        parsed_args = self._run_before_hooks(parsed_args)
        self.formatter = self._formatter_plugins[parsed_args.formatter].obj
        column_names, data = self.take_action(parsed_args)
        column_names, data = self._run_after_hooks(parsed_args,
                                                (column_names, data))
        self.produce_output(parsed_args, column_names, data)
        return 0


|

 .. image:: images/2week_04_table.png

|

 .. code-block:: python

    class Lister(display.DisplayCommandBase, metaclass=abc.ABCMeta):

        def produce_output(self, parsed_args, column_names, data):
            생략..

            self.formatter.emit_list(
            columns_to_include, data, self.app.stdout, parsed_args,
        )


|


 .. code-block:: python

    class TableFormatter(base.ListFormatter, base.SingleFormatter):

        ALIGNMENTS = {
            int: 'r',
            str: 'l',
            float: 'r',
        }

        def emit_list(self, column_names, data, stdout, parsed_args):
        x = prettytable.PrettyTable(
            column_names,
            print_empty=parsed_args.print_empty,
        )
        x.padding_width = 1

        # Add rows if data is provided
        if data:
            self.add_rows(x, column_names, data)

        min_width = 8
        self._assign_max_widths(
            stdout, x, int(parsed_args.max_width), min_width,
            parsed_args.fit_width)

        formatted = x.get_string()
        stdout.write(formatted)
        stdout.write('\n')
        return
            
