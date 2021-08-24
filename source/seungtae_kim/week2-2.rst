OpenStack 팀 2주차 진행사항 : openstack server list 명령어 동작 원리 파악
==========================================================

2주차 2번째 과제에 대한 풀이 진행 사항입니다.

1. 인자로 입력받은 server list 를 어떻게 구별해내는가?
2. server list  라는 명령어를 처리하는 파일은 무엇인가?
3. openstackcli 는 어떻게 nova api 주소를 알아내나요?
4. nova 의 어떤 API를 호출하여 결과를 받아오나요? ( 어떤 URI 를 호출하나요? )
5. 결과를 이쁘게 table 형식으로 출력해주는 함수는 무엇일까요?

1. 인자로 입력받은 server list 를 어떻게 구별해내는가?
---------

- 우선 순서대로 `openstackclient/shell.py` 에서 명렁어로 `server list` 라는 문구를 받게 되 파이썬 `sys` 모듈에 의해 `sys.argv[1:]`  입력된 것을 처리하게 됩니다.

- 그리고 해당 입력 값을 `OpenStackShell().run(argv)` 에서 처리해주게 되는데, 이 클래스와 `run` 함수는 python-openstackclient 레포의 의존성 모듈인 `osc_lib/shell.py` 에서 실행되게 됩니다.

    .. code-block:: osc_lib/shell.py/OpenStackShell class
        def run(self, argv):
            ret_val = 1
            self.command_options = argv
            try:
                ret_val = super(OpenStackShell, self).run(argv) # run은 cliff 모듈의 app 클래스의 메소드임.
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

- 다만, 여기서 주목할 것이 있습니다. 5번째 라인의 `ret_val` 이라는 변수에 상독된 값에 `run` 메소드가 한 번 더 붙었습니다.

- 이는 cliff (Command Line Interface Formulation Framework)의 약자로 커맨드 명령어를 빌드해주는 파이썬 프레임워크입니다.

- 해당 프레임워크의 app.py로 넘어가 확인하게 되면, 코드의 역할을 명확하게 다 이해하는 것은 아니지만 run 함수가 마찬가지로 있는 것을 볼 수 있습니다.

    .. code-block:: cliff/app.py
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
                    # When help is requested and `remainder` has any values disable
                    # `deferred_help` and instead allow the help subcommand to
                    # handle the request during run_subcommand(). This turns
                    # "app foo bar --help" into "app help foo bar". However, when
                    # `remainder` is empty use print_help_if_requested() to allow
                    # for an early exit.
                    # Disabling `deferred_help` here also ensures that
                    # print_help_if_requested will not fire if called by a subclass
                    # during its initialize_app().
                    self.options.deferred_help = False
                    remainder.insert(0, "help")
                self.initialize_app(remainder)
                self.print_help_if_requested()
            except Exception as err:
                if hasattr(self, 'options'):
                    debug = self.options.debug
                else:
                    debug = True
                if debug:
                    self.LOG.exception(err)
                    raise
                else:
                    self.LOG.error(err)
                return 1
            except KeyboardInterrupt:
                return _SIGINT_EXIT
            result = 1
            if self.interactive_mode:
                result = self.interact()
            else:
                try:
                    result = self.run_subcommand(remainder)
                except KeyboardInterrupt:
                    return _SIGINT_EXIT
            return result

- 그런데 이 코드에서 result라는 변수를 반환하고 있고, 해당 변수는 같은 클래스의 `run_subcommand`라는 함수를 호출하고 있습니다.

- 해당 함수는 cliff 모듈의 `command`의 run으로 다시 넘어가게 되는데, 왜 그런지는 모르겠으나 디버깅을 실행하게 되면 command의 run이 아닌 display의 run으로 이동하게 됩니다.

- 이게 왜 그런지 이해가 안되서 제출이 좀 늦었는데, 혹시 해답을 아시는 분이 있으면 알려주시면 감사하겠습니다.

- 이후 cliff/display.py -> cliff/lister.py -> compute/v2/server.py -> cliff/formatters/table.py를 지나서 결과 값이 출력되는 것 같습니다.

2. server list  라는 명령어를 처리하는 파일은 무엇인가?
---------

- `openstackclient/compute/v2/server.py` 의 `class ListServer(command.Lister)` 클래스에서 최종적으로 명령어를 처리하게 됩니다.

3. openstackcli 는 어떻게 nova api 주소를 알아내나요?
---------

- 위의 `compute/v2/server.py` 에서 `from novaclient.v2 import servers` -> `self.api.client.get("/servers/%s/tags" % base.getid(server))`

- 를 통해 nova api 주소를 찾아내는 것 같습니다.

4. nova 의 어떤 API를 호출하여 결과를 받아오나요? ( 어떤 URI 를 호출하나요? )
---------

- novaclient.v2의 servers 파일에서 `serverManager` 의 `_boot` 함수를 호출해 결과를 받아오는 것 같습니다.

- (디버깅 툴이 익숙치 않아서 수동으로 찾아보느라 시간이 많이 걸려서, 정확한 답인지 모르겠습니다...)

- from novaclient.v2 import servers

5. 결과를 이쁘게 table 형식으로 출력해주는 함수는 무엇일까요?
---------

- `cliff/formatters/table.py` 에서 테이블 형태로 만들어 출력을 지원하는 것 같습니다.