Week 2. openstack server list 명령어 동작 원리 파악
========================================================

openstack server list 는 환경변수에 등록된 프로젝트의 모든 인스턴스를 출력하는 명령어입니다.

python-openstackcli 의 코드 내부에서 이 명령어를 처리하기 위한 절차를 분석 후 자유롭게 보고서로 작성해주세요.

Index
--------------
- `1. 인자로 입력받은 server list 를 어떻게 구별해내는가?`_
- `2. server list 라는 명령어를 처리하는 파일은 무엇인가?`_

1. 인자로 입력받은 server list 를 어떻게 구별해내는가?
-----------------------------------------------------------------------------
인자와 명령어로 구분되고 명령어에 맞는 클래스를 매칭해주면서 server list라는 명령어를 구분해낸다.

1) 인자로 입력된 ['server', 'list']은 parser로 인자와 명령어로 구분된다.
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

.. code-block:: python

    # cliff/app.py/App
    class App(object):
        def run(self, argv):
            try:
                self.options, remainder = self.parser.parse_known_args(argv) # argv 는 options과 remainder로 분리된다.

2) OpenstackShll.command_manager의 find_command를 통해 명령어에 맞는 클래스로 연결된다.
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

.. code-block:: python

    # cliff/app.py/App
    class App(object):
        def run(self, argv):
            ...
            if self.interactive_mode:
                result = self.interact()
            else:
                try:
                    result = self.run_subcommand(remainder) # remainder = ["server", "list"]
                except KeyboardInterrupt:
                    return _SIGINT_EXIT
            return result

        def run_subcommand(self, argv):
            try:
                subcommand = self.command_manager.find_command(argv) # argv = ["server", "list"]

|

===============================================================================

2. server list  라는 명령어를 처리하는 파일은 무엇인가?
----------------------------------------------------------------

- server list 라는 명령어를 처리하는 파일은 앞서 말했던 `subcommand = self.command_manager.find_command(argv)` 의 find_command 에 답이 있다.
- 결과적으로 **python-openstackclient/openstackclient/compute/v2/server.py** 파일이 처리하게 되고 그 중에서도 **ListServer 라는 클래스** 에서 처리한다.


.. code-block:: python

    # cliff/app.py/App
    class App(object):
        def run_subcommand(self, argv):
            try:
                subcommand = self.command_manager.find_command(argv) # argv = ["server", "list"]

|

------------------------------------------------------------------------------------------

그렇다면 command_manager는 find_command를 어떻게 수행할까
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

- find_command의 동작을 확인해보자
- find_command는 단순하게 동작된다. argv를 name으로 치환해 그 argv name이 self.commands에 존재하면 commands의 value인 처리하는 클래스를 꺼내준다.

.. code-block:: python

    # Cliff.commandmanager.py
    class CommandManager(object):
        def find_command(self, argv):
            for i in range(start, 0, -1):
                name = ' '.join(argv[:i]) # argv[:i] : ['server', 'list']
                search_args = argv[i:]
                ...
                # Convert the legacy command name to its new name.
                if name in self._legacy:
                    name = self._legacy[name]

                found = None
                if name in self.commands:
                    found = name
                ...
                if found:
                    cmd_ep = self.commands[found]


------------------------------------------------------------------------------------------

self.command는 어떻게 만들어질까?
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
- self.command_manager 의 self는 처음 프로그램이 실행되는 python-openstackclient의 OpenstackShell이다.
- command_manager 는 OpenstackShell이 선언될 때 **osc_lib.command.commandmanager.CommandManager('openstack.cli')** 객체로 이루어진다.

.. code-block:: python

 from osc_lib.command import commandmanager

    # python-openstackclient/openstackclient/shell.py
    class OpenStackShell(shell.OpenStackShell):

        def __init__(self):

            super(OpenStackShell, self).__init__(
                description=__doc__.strip(),
                version=openstackclient.__version__,
                command_manager=commandmanager.CommandManager('openstack.cli'),
                deferred_help=True)

|

- osc_lib.command.commandmanager.CommandManager('openstack.cli') 실행부는 Cliff.commandmanager로 이어진다.

.. code-block:: python

    # Cliff.command.commandmanager.py
    class CommandManager(object):
        def __init__(self, namespace, convert_underscores=True):
            self.commands = {}
            self._legacy = {}
            self.namespace = namespace
            self.convert_underscores = convert_underscores
            self.group_list = []
            self._load_commands() # 주목

- 여기서 주목해볼 것은 commandManager의 _load_commands()이다.
- _load_commands는 각 패키지의 namespace를 순회하며 load_commands를 호출한다.
- load_commands는 **stevedore.ExtensionManager** 를 통해 패키지에 대한 정보를 가져와서 commands에 각 명령어에 맞는 entry_point를 등록한다.

.. code-block:: python

    # Cliff.command.commandmanager.py
    def _load_commands(self):
        if self.namespace: # 패키지 네임을 순회하면서
            self.load_commands(self.namespace)

    def load_commands(self, namespace):
        """Load all the commands from an entrypoint"""
        self.group_list.append(namespace)
        for ep in stevedore.ExtensionManager(namespace): # stevedore.ExtensionManager에서 패키지에 대한 정보를 가져와서
            LOG.debug('found command %r', ep.name)
            cmd_name = (ep.name.replace('_', ' ')
                        if self.convert_underscores
                        else ep.name)
            # 여기서 cmd_name은 "server list" 처럼 문자열로 된 명령어이고
            # entry_point는 <openstackclient.compute.v2.server.ListServer object>와 같이 명령어를 처리할 ListServer이다.
            self.commands[cmd_name] = ep.entry_point # commands에 entry_point에 등록해준다.


- commandManager의 commands list는 패키지 이름별로 stevedore.ExtensionManager를 이용해 읽어와 명령어와 수행할 클래스를 매칭하여 만들어진다.

.. note::

    여기서부터는 Openstack 코드를 벗어나 stevedore.ExtensionManager와 python package 동작에 대해 이야기합니다.
    어려우시면 다음으로 넘어가셔도 좋습니다.

------------------------------------------------------------------------------------------

stevedore.ExtensionManager는 어떻게 명령어를 처리할 클래스를 알고 있을까
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
- load_commands에서 stevedore.ExtensionManager(namespace)는 for loop로 실행되었다.
- python class에서 loop를 실행하는 함수는 __iter__이고 **self.extensions** 이라는 list 객체를 순회하는 간단한 로직이다.

.. code-block:: python

    #stevedore.extension.py
    class ExtensionManager(object):
        ...
        def __iter__(self):
            return iter(self.extensions)

- self.extensions가 어떻게 선언되는지 알기 위해 ExtensionManager의 __init__을 보자
- self.extensions은 결국 self._load_plugins 함수의 리턴 값이다.
- self._load_plugins은 self.list_entry_points로 가져온 entry_points를 _load_one_plugin으로 읽어내는 로직이다.

.. code-block:: python

    class ExtensionManager(object):
        def __init__(self, namespace, ...):
            ...
            extensions = self._load_plugins(invoke_on_load,
                                            invoke_args,
                                            invoke_kwds,
                                            verify_requirements)
            self._init_plugins(extensions)

        def _init_plugins(self, extensions):
            self.extensions = extensions # self.extensions은 결국 self._load_plugins 함수의 리턴 값이다.
            self._extensions_by_name_cache = None

        def _load_plugins(self, invoke_on_load, invoke_args, invoke_kwds,
                  verify_requirements):
            extensions = []
            for ep in self.list_entry_points(): # self.list_entry_points로 entry_points 값을 가져와서
                try:
                    # self._load_one_plugin : self.list_entry_points를 읽은 값을 load하는 함수
                    ext = self._load_one_plugin(ep,
                                                invoke_on_load,
                                                invoke_args,
                                                invoke_kwds,
                                                verify_requirements,
                                                )
                    if ext:
                        extensions.append(ext)
                ...
            return extensions

- extensions은 list(stevedore._cache.get_group_all(self.namespace))을 불러와 추가해서 만들어진다.
- **stevedore._cache.get_group_all(self.namespace)를 보면 self._get_data_for_path(path)를 통해 데이터를 불러오고 importlib_metadata.EntryPoint(*vals)로 데이터를 읽어들여 return 하는 형태이다.**
- self._get_data_for_path는 프로젝트 setup 때 프로젝트 path hash 값으로 생성한 python local App data 파일을 불러온다. 그 속엔 패키지별로 컴포넌트 이름, 값, 컴포넌트 그룹이 저장되어 있다. 보통 윈도우에는 App data / local / Python Entry Point에 저장되어 있다.

.. code-block:: python

    # stevedore/_cache.py
    class Cache:
        def __init__(self, cache_dir=None):
            if cache_dir is None:
                cache_dir = _get_cache_dir() # 내 컴퓨터의 local App data의 python Entry Points 폴더
            self._dir = cache_dir
            ...

        def _get_data_for_path(self, path):
            ...
            digest, path_values = _hash_settings_for_path(path) # python setup 할 때 클래스 룩업 캐쉬로 연결 (import 할 이름 : 클래스 정식 이름, 실패 했을 때)
            filename = os.path.join(self._dir, digest) # Python Entry Points / 클래스 룩업 캐쉬
            try:
                ...
                with open(filename, 'r') as f:
                    data = json.load(f)
            ...
            return data

        def get_group_all(self, group, path=None):
            result = []
            data = self._get_data_for_path(path) # 프로젝트 setup 때 프로젝트 path hash 값으로 생성한 python local App data / entry point 정보을 불러온다.
            group_data = data.get('groups', {}).get(group, [])
            for vals in group_data:
                result.append(importlib_metadata.EntryPoint(*vals)) # val은 패키지에 대한 name value group 정보
            return result

- 앞서 self._get_data_for_path로 받아온 패키지의 컴포넌트 이름 / 값 / 컴포넌트 그룹이 data에 담겨있었다.
- 그 후 data는 vals 변수로 순회하며 importlib_metadata.EntryPoint(\*vals)를 수행해 리턴할 결과값을 만들게 된다.
- importlib_metadata는 importlib/metadata.py의 EntryPoint 값으로 패키지 setup 시에 package_name.egg-info에 저장된 프로젝트 이름, wheel 정보, 모듈 로딩한 엔트리포인트 정보를 읽어오게 된다.
- package_name.egg-info에 저장된 entry_points.txt 정보를 읽어서 vals로 data에서 받은 name, value, group 정보에 부합하는 부분을 리턴하게 된다.

.. code-block:: python

    # importlib/metadata.py
    @property
    def entry_points(self):
        return EntryPoint._from_text(self.read_text('entry_points.txt'))

    @classmethod
    def _from_text(cls, text):
        ...
        return EntryPoint._from_config(config)

    @classmethod
    def _from_config(cls, config):
        return [
            cls(name, value, group)
            for group in config.sections()
            for name, value in config.items(group)
            ]

- stevedore.ExtensionManager는 python setup 시에 읽어낸 entry point를 활용해 읽어들인 값을 통해 명령어 이름과 수행 클래스를 알아낸다. 기본적인 클래스의 내용은 package_name.egg-info의 entry_point.txt에 저장되어 있다.

.. note::

    classmethod에 사용하는 cls를 잘 모르겠다면 https://firework-ham.tistory.com/101

- 더 나아가, package_name.egg-info에 구성되는 entry_points.txt는 setup 방식 중 pbr을 사용해서 만들어진다.
- pbr은 setuptools 패키징을 관리하는 방식 중 하나로 setup.cfg를 사용해 세밀한 entry_points를 만들 수 있는 것이 특징이다. 자세한 설명은 https://docs.openstack.org/pbr/latest/ 를 참고해주길 바란다.

.. code-block:: python

    import setuptools

    setuptools.setup(
        setup_requires=['pbr>=2.0.0'],
        pbr=True)

|

