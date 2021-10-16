====================================================
2week - openstack server list 명령어 동작 원리 파악
====================================================

1. 인자로 입력받은 server list 를 어떻게 구별해내는가?
shell.py의 main은 아래의 코드와 같다.

.. code-block:: python

    def main(argv=None):
        if argv is None:
            argv = sys.argv[1:]

        return OpenStackShell().run(argv)


run의 인자로는 argv가 주어지는데, 이 argv를 알아보고자 중간에 print(args)를 삽입하여 출력해본 결과, ['server', 'list']라는 것을 알 수 있다.
run 메소드가 해당하는 ../osc-lib/shell.py의 run으로 해당하는 것을 볼 수 있다.

아래는 ../osc-lib/shell.py의 run()의 코드!

.. code-block:: python

    def run(self, argv):
        ret_val = 1
        self.command_options = argv
        try:
            ret_val = super(OpenStackShell, self).run(argv)
            return ret_val
        except Exception as e:
            if not logging.getLogger('').handlers:
                logging.basicConfig()
            if self.dump_stack_trace:
                self.log.error(traceback.format_exc())
            else:
                self.log.error('Exception raised: ' + str(e))

            return ret_val

        finally:
            self.log.info("END return value: %s", ret_val)


../osc-lib/shell.py내의 OpenStackShell 클래스를 인자로 사용하여 부모 상속자(OpenStackShell)의 run() 메소드를 실행하며 args(server, list) 인자를 넘겨주는 것을 볼 수 있다.
아래는 cliff.app.App의 run() 메소드의 코드이다.

.. code-block:: python

    def run(self, argv):
        """Equivalent to the main program for the application.

        :param argv: input arguments and options
        :paramtype argv: list of str
        """
        try:
            self.options, remainder = self.parser.parse_known_args(argv)
            self.configure_logging()
            self.interactive_mode = not remainder
            """생략"""
        else:
            try:
                result = self.run_subcommand(remainder)
            except KeyboardInterrupt:
                return _SIGINT_EXIT
        return result


args를 인자로 받고 run() 메소드를 실행하는데, `result = self.run_subcommand(remainder)` 에서 값을 처리한다.
run_subcommand() 메소드의 코드를 분석해보면, argv를 인자로 받아서 `subcommand = self.command_manager.find_command(argv)` 를 사용하여 find_command 메소드를 사용하게 된다.

.. code-block:: python

    def find_command(self, argv):
        """Given an argument list, find a command and
        return the processor and any remaining arguments.
        """
        start = self._get_last_possible_command_index(argv)
        for i in range(start, 0, -1):
            name = ' '.join(argv[:i])
            search_args = argv[i:]
            # The legacy command handling may modify name, so remember
            # the value we actually found in argv so we can return it.
            return_name = name
            # Convert the legacy command name to its new name.
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
                    # NOTE(dhellmann): Some fake classes don't take
                    # require as an argument. Yay?
                    arg_spec = inspect.getfullargspec(cmd_ep.load)
                    if 'require' in arg_spec[0]:
                        cmd_factory = cmd_ep.load(require=False)
                    else:
                        cmd_factory = cmd_ep.load()
                return (cmd_factory, return_name, search_args)
        else:
            raise ValueError('Unknown command %r' %
                             (argv,))


find_command() 메소드를 통해서, server list를 구분하는 것으로 유추해보았다.

2. server list  라는 명령어를 처리하는 파일은 무엇인가?
.. image:: ../images/week2/week2-2-2.png

'../python-openstackclient/openstackclient/compute/v2/server.py' 파일에서 server list를 처리한다.
또한 ListServer 클래스를 사용해서 명령어를 사용하는 것으로 유추해볼 수 있다.

3. nova 의 어떤 API를 호출하여 결과를 받아오나요? ( 어떤 URI 를 호출하나요? )
ListServer 클래스의 take_action() 메소드를 주목해보면 project_id, user_id, flavor_id 등 server list 명령어를 시행 시 관련된 정보들을 take_action() 메소드를 이용해서 가져오는 것을 알 수 있다.
또한, 이에 대한 정보들을 list로 가져오는 것을 novaclient.v2.client의 servers 메소드에서 가져오는데 이는 `self.servers = servers.ServerManager(self)` 을 실행하게 된다.
servers.py의 ServerManaver 클래스를 보게되면 list 메소드가 있다
아래는 list 메소드 코드이다.

.. code-block:: python

        def list(self, detailed=True, search_opts=None, marker=None, limit=None,
             sort_keys=None, sort_dirs=None):
        """생략"""
        detail = ""
        if detailed:
            detail = "/detail"

이처럼 '/detail' url을 보여주는데, 'server/detail' url을 호출하여 결과를 받아오는 것으로 유추해보았다.
