2week - Openstack server list 명령어 동작 원리 파악
==========================================================

2주차 과제는 우리가 요청한 openstack server list의 값을 콘솔에 표시하기 까지 어떤 과정이 일어나는지
알아내는 과제였습니다. 워낙 방대한 파일이기 때문이기에 디버깅 하기에 다소 부족한 면이 있었습니다.

디버깅 툴로는 pycharm과 sublime text를 사용했습니다. pycharm의 경우 break point를 사용한
디버깅이 가능하다는 장점이 있었고 sublime text는 단어 검색에 빠르다는 장점이 있었습니다.


0. 디버깅 전 알면 좋은 팁
_________________________________________________________________________________

 1. run 함수는 cliff의 함수를 상속하고 있다.  (부모) cliff -> osc.lib -> shell.py 와 같은 형식으로 상속 관계를 가지고 있음.


1. 인자로 입력받은 server list를 어떻게 구별해내는가? 
______________________________________________________________________________________________

 1. 우선 가장 먼저 파일을 실행하는 python-openstackclient/openstackclient/shell.py에서 디버깅을 시작하였습니다.

    .. code-block::
    
        def main(argv=None):
          if argv is None:
            argv=sys.argv[1:]

   위 사진과 같이 shell.py 는  main에서 OpenstackShell().run(argv) 를 실행하게 됩니다.
   이때 argv 로 리스트 형태로 ['server', 'list'] 값을 넘겨주게 됩니다.

 2. run 함수의 위치를 찾아가 보면 osc-lib/osc-lib/shell.py 에서 해당 함수를 찾을 수 있습니다.
    이 함수에서도 run 함수를 실행하기에 해당 함수로 진입해봅니다.

    .. code-block::

        def run(self, argv):
            ret_val = 1
            self.command_options = argv
            try:
                ret_val = super(OpenStackShell, self).run(argv)
                return ret_val

 3. run 함수를 실행하다보면  `initialize_app <https://github.com/openstack/python-openstackclient/blob/master/openstackclient/shell.py#L130>`_ 이라는 함수를 실행하고 있음을
    볼 수 있습니다.

    .. code-block::

        def initialize_app(self, argv):
            super(OpenStackShell, self).initialize_app(argv)

            # Re-create the client_manager with our subclass
            self.client_manager = clientmanager.ClientManager(
                cli_options=self.cloud,
                api_version=self.api_version,
                pw_func=shell.prompt_for_password,
            )

 4. initialize_app 함수는 openstackclient/shell.py 의 `_load_plugins <https://github.com/openstack/osc-lib/blob/master/osc_lib/shell.py#L374>`_ 함수에서 필요한 plugin를 설정해주고 있습니다. 여기서 api, cmd_group 과 같은 값들이
    설정됨을 확인 할 수 있습니다.

    .. code-block::

        def _load_plugins(self):
        """Load plugins via stevedore
        osc-lib has no opinion on what plugins should be loaded
        """
        # Loop through extensions to get API versions
        for mod in clientmanager.PLUGIN_MODULES:

        ...

        version = '.v' + version_opt.replace('.', '_').split('_')[0]
                cmd_group = 'openstack.' + api.replace('-', '_') + version
                self.command_manager.add_command_group(cmd_group)

    자세한 과정은 _load_plugins 함수를 보면 `clientmanager.PLUGIN_MODULES <https://github.com/openstack/python-openstackclient/blob/master/openstackclient/shell.py#L71>`_ 를 볼 수 있습니다. 이 PLUGIN_MODULES 을 어떻게 받아오는지 
    보면 get_plugin_modules(grop) 함수에서 stevedore.ExtensionManger(group) 을 통해 필요한 모듈을 저장함을 볼 수 있습니다.
    stevedore 는 동적으로 plugin을 불러오는 라이브러리이며 어떤 모듈을 불러올지는 python setup.py develop load시킨 setup.cfg를 참고하여 모듈을 불러온다고
    합니다. 정리하면 stevedore 를 통해 저장되어 있는 모듈들을 동적으로 읽어와서 이를 바탕으로 cmd_group 을 만들어준다는 것입니다.

    .. code-block::

        def get_plugin_modules(group):
        """Find plugin entry points"""
        mod_list = []
        mgr = stevedore.ExtensionManager(group) // stevedore 를 통한 mod plugin load

 5. 필요한 plugin을 load 한 후 비슷한 방법으로 load_command 를 실행합니다. 이 또한 command_manager에 있는 load_command가
    namespace를 인자로 넘겨주면 stevedore 를 통하여 명령어들을 load 하여 entry point를 저장해둡니다.

    ..code-block::

        def load_commands(self, namespace):
        """Load all the commands from an entrypoint"""
        self.group_list.append(namespace)
        for ep in stevedore.ExtensionManager(namespace): //stevedore 를 통한 command load
            LOG.debug('found command %r', ep.name)
            cmd_name = (ep.name.replace('_', ' ')
                        if self.convert_underscores
                        else ep.name)
            self.commands[cmd_name] = ep.entry_point // command 와 entry_point 를 key:Value 형태로 저장
        return

 6. 이렇게 initialize_app 함수가 끝나면 run_subcommand(remainder) 함수를 실행함을 볼 수 있습니다.
    이 함수를 따라가다보면 subcommand=self.command_manager.find_command(argv) 함수를 볼 수 있습니다.
    앞서 우리가 stevedore를 통해 load 해둔 명령어 중 sever list를 키로 하는 값이 있는지 찾는 함수 입니다. 이 함수의 반환값을 살펴보면
    cmd_factory, return_name, search_args를 반환하게 됩니다. 

    .. code-block::

        def find_command(self, argv):
  
            ...

             if found:
                cmd_ep = self.commands[found]  //앞서 stevedore에서 found(sever list) 값을 키로 하는 Value를 cmd_ep에 할당
                
                ...

                return (cmd_factory, return_name, search_args) //commander class 반환



2.server list 라는 명령어를 처리하는 파일은 무엇인가?
___________________________________________________________

 1. run 함수를 진행하다보면 prepare_to_run_command를 실행하게 됩니다. 여기서 cmd 실행 시 clientmanager 를 통해 인증이 필요한지 
    검사를 합니다.
    
    .. code-block::

        try:
             self.prepare_to_run_command(cmd) // cmd 실행시 인증이 필요한지 여부 검사
             full_name = (cmd_name
                         if self.interactive_mode
                         else ' '.join([self.NAME, cmd_name])
                         )
             cmd_parser = cmd.get_parser(full_name)
             try:
                parsed_args = cmd_parser.parse_args(sub_argv)
             except SystemExit as ex:
                raise cmd2.exceptions.Cmd2ArgparseError from ex
             result = cmd.run(parsed_args) // 명령어 실행

 2. cmd.run(parsed_args) 을 통해 해당 명령어를 처리하는 듯합니다. run함수에는 self.take_action(parsed_args)를 
    실행합니다. 여기서 실질적인 api 통신이 이루어지며 nova로 부터 결과를 받아옵니다. 
    take_action은 server.py 에 위치하여 있지만 해당 파일에 여러개의 take_action이 존재함을 볼 수 있습니다(overloading).
    따라서 인자로 넘겨준 parsed_args 에 해당하는 take_action 만을 실행시킨다는 점이 중요한 것 같습니다.

    .. code-block::

        def run(self, parsed_args):
        parsed_args = self._run_before_hooks(parsed_args)
        self.formatter = self._formatter_plugins[parsed_args.formatter].obj
        column_names, data = self.take_action(parsed_args) // 해당하는 인자의 take_action을 한 후 res 데이터를 저장 
        column_names, data = self._run_after_hooks(parsed_args,
                                                   (column_names, data))
        self.produce_output(parsed_args, column_names, data) // 출력 함수
        return 0

3. openstackcli는 어떻게 nova api 주소를 알아내나요?
___________________________________________________________

 1. 앞선 과정에서 sever list에 해당하는 정보를 얻게 되면 server.py에 있는 여러 take_action 에서 인자에 맞는 take_action이 실행되게 됩니다.
    이 과정에서 우리는 sever list에 해당되는 정보를 가지고 있기 때문에 해당 파일을 담당하는 servers.py 의 list 함수를 실행하게 되고
    여기서 해당 url을 생성해줍니다. (header 정보는 parser 설정에서 완료)

    .. code-block::

         def list(self, detailed=True, search_opts=None, marker=None, limit=None,sort_keys=None
                    ,sort_dirs=None):
            if detailed:
                detail = "/detail"  // detail   설정 여부에 따라 url에 /detail 추가

             ...

             servers = self._list("/servers%s%s" % (detail, query_string), //결국 /severs/detail 이라는 인자를 넘겨줌
                                 "servers")


4. nova의 어떤 API를 호출하여 결과를 받아오나요?
___________________________________________________________
 
 1. 3번 과정에서 /servers/detail 이라는 url이 생성됩니다. novaclient/base.py 에서는 이를 인자로 하는 api.client.get(url)
    을 실행하게 되며 이 함수에서는 request 함수를 실행하게 됩니다.

    .. code-block::

         def _list(self, url, response_key, obj_class=None, body=None,
              filters=None):
         if filters:
            url = utils.get_url_with_filter(url, filters)
         if body:
            resp, body = self.api.client.post(url, body=body)
         else:
            resp, body = self.api.client.get(url) // sever list 는 get을 사용하므로 해당 코드 실행

 2. venv/Lib/site-packages/keystoneauth1-4.3.1-py3.9.egg/keystoneauth1/adapater.py 의 
    request(args, kwargs) 함수를 통해 request를 완료하고 해당 값을 저장합니다.
   
    .. code-block::

     def request(self, *args, **kwargs):
        headers = kwargs.setdefault('headers', {})
        headers.setdefault('Accept', 'application/json')

         try:
            kwargs['json'] = kwargs.pop('body')
         except KeyError:
            pass

        resp = super(LegacyJsonAdapter, self).request(*args, **kwargs) // 실제 api 통신 후 값을 받아오는 곳



 3. 또한 server.py에서 image_client.images() compute_client.flavor  함수를 통해서
    images/detail , flavors/detail?is_public api도 호출함을 볼 수 있습니다.
 
    .. code-block::
    
         try:
            images_list = image_client.images() // /image api 호출
                for i in images_list:
                    images[i.id] = i
         except Exception:
                pass

         ...

         try:
            flavors_list = compute_client.flavors.list(is_public=None) //  /flavor/detail api 호출


5. 결과를 이쁘게 table 형식으로 출력해주는 함수는 무엇일까요?
_________________________________________________________________________________
 1. 우선적으로 출력을 담당하는 함수는 produce_ouput(parsed_args, column_names,data) 함수입니다.

    .. code-block::

         def run(self, parsed_args):
         
         ...

         self.produce_output(parsed_args, column_names, data) // 출력 함수
         return 0


 2. 1번 함수를 따라가보면 python-openstackclient/venv/Lib/site-packages/cliff-3.8.0-py3.9.egg/cliff/formatters/table.py
    의 emit_list 에서 출력 값 형식을 생성함을 볼 수 있습니다.

    .. code-block::
        
        self.formatter.emit_list(
            column_to_include, data, self.app.stdout, parsed_args,
            )

 3. emit_list 내부 함수인 88번줄의 prettytable.prettyTable 과 96번줄의 self.add_rows 둘 중 하나의 함수로 예측하였습니다.
    prettytable.prettyTable 을 실행한 후 x 의 값은 prettyTable 객체가 담겨 있으며 이 부분이 기본적인 틀 역할을 해주며
    실제 원하는 값들은 add_rows를 통해 이 틀 안에 데이터를 삽입해주는 것 같습니다. 
    
    .. code-block::

         def emit_list(self, column_names, data, stdout, parsed_args):
             x = prettytable.PrettyTable(
                column_names,
                print_empty=parsed_args.print_empty,
             )
             x.padding_width = 1

             # Add rows if data is provided
             if data:
                self.add_rows(x, column_names, data)

         …

             formatted = x.get_string()
             stdout.write(formatted)
             stdout.write('\n')
             return

 4. 이렇게 출력 형식이 지정되면 formatted= x.get.string() 과 stdout 함수를 통해 사용자의 콘솔에 출력함을 볼 수 있습니다.