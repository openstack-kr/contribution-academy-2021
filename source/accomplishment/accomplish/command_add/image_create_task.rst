openstackclient에 image create task 명령어 추가하기
=====================================================================

1. 커멘드 구현에 앞서
------------------------------
 현재 openstackcli에서 glance 관련 함수를 수행하기 위한 적절한 커맨드가 존재하지 않습니다.
 따라서 저희는 아직 구현되지 않은 glance 관련 함수를 조사 하여 문서를 만들었습니다. 
 `(참고) <https://docs.google.com/spreadsheets/d/1AMu9tqF3HPXwyapc7RtcUR-PfQP7q2o2cvozNpCS0yg/edit#gid=0>`_

 이 글은 해당 문서에서 task create 를 구현하기 위한 과정이 담겨있습니다!   

--------

2. glance 에서 task create 확인하기
--------------------------------------------------------

 처음에는 해당 명령어가 어떤 기능을 하고 있는지 살펴보기 위해 구축된 devstack 에서 해당 명령어를 입력해봤습니다.
 그럼 어떤 명령어를 입력해야하나를 알아야 했습니다. 이는 ``glance api`` 문서에 친철하게 있기 때문에 다음과 같은 
 코드로 task create가 되는지 확인 해보았습니다. 

 .. code-block:: powershell
    
     $ image task create --type import  --input  "{'import_from': 'http://app-catalog.openstack.example.org/groovy-image',
      'import_from_format': 'qcow2','image_properties': {'disk_format': 'vhd','container_format': 'ovf'}}"
 
 .. image:: images/create_task_1.png
        :height: 300
        :width: 700


 코드는 잘 돌아간다!!!! 그러면 이제 뭘 하지... 라는 생각이 들었습니다.

3. python-glanceclient 를 clone 해서 살펴보기
-----------------------------------------------------------------

    1.  task 파일 분석

        python-glanceclient 를 clone 받고나서 코드들을 쭉 살펴보았다. 폴더 구조를 보면 openstackclient 처럼 
        v2 라는 폴더를 볼 수 있고 안 쪽에는 image 관련 함수들이 있다. 
        여기서 `task <https://github.com/openstack/python-glanceclient/blob/master/glanceclient/v2/tasks.py>`_
        라는 것이 있으니 한번 살펴보았다. 처음 코드를 봤을 때는 누구나 이게뭐지.... 할 것이다. 그나마 알겠는건
        /v2/tasks 를 post 하고 있으니 이 부분에서 API를 호출하겠구나 짐작해 볼 수 있었다.
 
 
    2. shell.py 파일 분석

        그럼 이 전체 함수를 진행하는 main 느낌의 
        `shell.py <https://github.com/openstack/python-glanceclient/blob/master/glanceclient/v2/shell.py#L1507>`_
        을 살펴보자! 여기서는 API 에서 명세된 매개변수가 써있고 
        우리가 앞서 봤던 task 파일의 함수를 실행시키는 함수가 있습니다.
        
        .. code-block:: python

            @utils.arg('--type', metavar='<TYPE>',   # glance api 에서 봤던 매개변수가 있다.
                help=_('Type of Task. Please refer to Glance schema or '
                      'documentation to see which tasks are supported.'))
            @utils.arg('--input', metavar='<STRING>', default='{}',
                help=_('Parameters of the task to be launched'))

            def do_task_create(gc, args):  # 이 함수를 실행해서 task를 만들겠구나!
                """Create a new task."""
                if not (args.type and args.input):
                    utils.exit('Unable to create task. Specify task type and input.')
            else:
                try:
                    input = json.loads(args.input)
                except ValueError:
                    utils.exit('Failed to parse the "input" parameter. Must be a '
                               'valid JSON object.')

                task_values = {'type': args.type, 'input': input}  # 매개변수를 저장하고
                task = gc.tasks.create(**task_values)              # 앞서 봤던 task 파일의 함수를 실행하는 듯하다!
                ignore = ['self', 'schema']
                task = dict([item for item in task.items()
                         if item[0] not in ignore])
                utils.print_dict(task)

        아직은 봐도 무슨 말인지 모르는게 너무 많을 겁니다. 사실 처음 이 코드를 봐도 허탕 쳤다라는 생각이 들었고
        openstackclinet 로 넘어가야겠다는 생각을 하게 되었습니다.

4. openstackclient 살펴보기
-----------------------------------------------
    1.  배웠던 setup.cfg 를 살펴보자! 

        우리가 앞서 openstack server list 함수를 분석했을때 명령어가 setup.cfg에 명시되어 있어 
        load.command 함수를 통해 mapping 함을 볼 수 있었습니다. 이에 ``image create task`` 라는 
        명령어를 만들기 위해서는 일단 해당 명령어를 쓸 수 있게 setup.cfg에 이를 명시해주는게 우선시
        되어야 생각했습니다.

     .. code-block:: python

        openstack.image.v2 =
            image_add_project = openstackclient.image.v2.image:AddProjectToImage
            image_create = openstackclient.image.v2.image:CreateImage
            image_delete = openstackclient.image.v2.image:DeleteImage
            image_list = openstackclient.image.v2.image:ListImage
            image_member_list = openstackclient.image.v2.image:ListImageProjects
            image_remove_project = openstackclient.image.v2.image:RemoveProjectImage
            image_save = openstackclient.image.v2.image:SaveImage
            image_show = openstackclient.image.v2.image:ShowImage
            image_set = openstackclient.image.v2.image:SetImage
            image_unset = openstackclient.image.v2.image:UnsetImage
            image_task_create = openstackclient.image.v2.image:CreateTask // 제가 추가 해준 코드입니다.

    
    2.  CreateTask Class 만들어 주기

        위에 코드에서 전부  ``openstackclient.image.v2.image`` 라는 경로를 참조하고 있습니다. 
        그러면 : 뒤에는 뭐를 나타내고 있을까요? 물론 코드에 능숙하신분은 Class 라는걸 바로 눈치채셨겠지만 
        한번 확인해보는 것이 좋겠죠? 해당 경로로 넘어가기 전에 ``ShowImage``, ``ListImage`` 처럼 
        : 뒤에 있는 글자들을 기억하고 넘어가도록 합시다.
        

        이렇게 넘어오면 예상한대로 Class 가 존재하고 있습니다. L1147 를 보면 ShowImage 가 있네요!
        그러면 여기서 우리는 CreateTask 라는 클래스를 만들어줘야한다는 목표가 생깁니다. 일단은 Create
        라는 공통점이 있는 CreateImage 클래스를 참고해 다음과 같이 만들수 있었습니다.
      
        .. code-block:: python

             class CreateTask(command.ShowOne):
                _description = _("Create a new task from attributes") //

                def get_parser(self, prog_name):
			
                    parser = super(CreateTask, self).get_parser(prog_name)
                    return parser

                 def take_action(self, parsed_args):
       
                     return zip(*sorted(info.items()))
       

5. command 구현하기
----------------------------------------------------
      
    1. get_parser

        모든 클래스를 보면 get_parser 와 take_action 을 볼 수 있습니다. 다른 코드들을 참고한 결과,
        get_parser 의 경우 glance API 에 request 를 보낼 때 필요한 인자들이 담겨져 있음을 확인 할 수 있습니다. 
        그렇다면 저같은 경우에는 두 개의 인자 (input, type) 을 명시해주면 된다는 생각을 하게 되었습니다.

        .. code-block:: python

            def get_parser(self, prog_name):
			
                    parser = super(CreateTask, self).get_parser(prog_name) // 기본적으로 있어야할 parser 객체 생성
                    parser.add_argument(
                        "--type",            // type 인자 추가
                        metavar="<type>",
                        help=_("Type of Task. Please refer to Glance schema or "
                        "documentation to see which tasks are supported."),
                    )
                    parser.add_argument(
                        "--input",           // input 인자 추가
                        metavar="<STRING>",
                        default='{}',
                        help=_("Parameters of the task to be launched."),
                    )
                    return parser


        이렇게 설정해주면 아 이 함수는 type , input 이라는 함수가 필요해!! 라고 선언하는 것이 됩니다. 
        여기서 고민했던 부분은 `metavar, default 이런건 뭘 추가해줘야하지?` 였습니다. 이 해답은 
        위의 glanceclient의 shell.py 코드를 살펴보니깐 답이 나왔습니다!! @utils 을 그래로 
        옮겨 주시면 됩니다. 삽질 했다고 느꼈던 glance 코드분석이 이렇게 도움이 되었습니다. 
        
    2. take_action

        take_action은 이름을 보면 실제 행동을 하는 함수겠죠? 그러면 여기서 create task
        의 동작을 구현해줘야된다고 생각했습니다. 이 부분 때문에 sever list, image list 등
        결과값을 쉽게 볼 수 있고 그나마 알고 있는 함수들을 디버그 하면서 이해하려고 했습니다. 
        그래도 이걸 어떻게 구현하지 라는 생각밖에 들지 않았습니다. 그렇게 코드 분석을 하는 중 
        같은 glance 쪽 command를 구현하고 계시는 임승현 멘티님이 
        `proxy <https://github.com/openstack/openstacksdk/blob/master/openstack/image/v2/_proxy.py>`_ 
        파일을 찾으셨습니다! 이 파일을 보면 다음과 같이 task_create 라는 함수가 존재했습니다.

        .. code-block:: python

                def get_parser(self, prog_name):
			
                    return self._create(_task.Task, **attrs)




        코드를 살펴보면 create 라는 함수를 실행하는데 이를 계속 들어가면 이 함수는 최종적으로
        glance API 에 요청을 보내는 것을 알 수 있습니다. 즉 glance API에 task를 만들어주세요
        요청만 보내면 되는거고 따로 task를 만들기 위해 고민할 필요는 없었습니다.
        자 그럼 어떻게 요청을 보내고 값을 받아 올지의 관점으로 코드를 짜보겠습니다.
        image create 와 비교해보면서 보시면 좋습니다!!

        .. code-block:: python

             def take_action(self, parsed_args):
                
                image_client = self.app.client_manager.image  # image_client.create_task를 사용하기 위해 해당 코드가 필요
                task = image_client.create_task(**kwargs)     # 매개변수들을 전달하여 glance API에 요청
        
                return zip(*sorted(info.items()))  


        우선 이렇게 기본적으로 api를 요청 해주세요 라는 코드를 작성했습니다. zip의 의미는 모르겠으나
        일단 image create 에서도 이렇게 했으니 그대로 따라 해보았습니다.

    3. 함수 완성시키기

     1. take action 완성하기

       앞서 틀만 잡아주었던 take action을 image create 함수를 참고하여 내용을 추가했습니다.

        .. code-block:: python

         def take_action(self, parsed_args):
            kwargs = {}   # API 요청 떄 보낼 매개변수
            info = {}     # 화면에 출력 될 값을 저장 (API 요청 결과를 담을 예정)
            image_client = self.app.client_manager.image 
            copy_attrs = ('type', 'input')  # 매개변수에서 우리가 따로 API 한테 보내줄 내용
            for attr in copy_attrs:         # 우리가 필요한 매개변수가
                if attr in parsed_args:     # 매개변수를 저장한 클래스에 존재하나요?!
                    val = getattr(parsed_args, attr, None)  # 존재한다면 그 값을 넘겨주세요
                if val:
                    # Only include a value in kwargs for attributes that
                    # are actually present on the command line
                     kwargs[attr] = val   # 그리고 그값을 kwargs 에 저장해둘게요

            task = image_client.create_task(**kwargs)   # 매개변수를 넘겨주면서 glance api를 요청할게요!
            if not info:
                info = _format_image(task) # 받아온 값을 출력할 수 있도록 형식을 만들어주세요!
            return zip(*sorted(info.items())) # 형식과 값들을 잘 정리해서 최종 결과를 넘겨줍니다.


        이렇게 제가 이해한대로 주석을 달고 디버그를 진행했습니다. 그 결과 두가지의 문제가 생겼습니다.
        (왜 코딩은 한번에 성공할 수 없는 것인가..!)

        *   우선 main 함 수에서 인자를 구분할 때 \" \" 를 기준으로 하기 때문에 
            앞서 devstack에 입력한 명령어를 그대로 입력하면 \"import_from_format\": \"qcow2"\ 와 같은 key-value 
            형식을  import_from_format,  qcow2 처럼 각기 다른 값으로 
            저장해버리는 문제가 발생하였습니다. 따라서 명령어 입력시에 \' 와 \" 를 
            서로 반전해서 넣어주는 방법으로 인자를 전달해주어야 합니다. 이때 값이 넘어가면서 또 input 값에 공백값,
            \'\n\' 과 같은 개행문자가 들어가는 현상이 나타나서 replace 명령어로 이를 없애주는 작업도 필요 했습니다.

        *   두번째로 생긴 문제는 ``task = image_client.create_task(**kwargs)`` 에서 
            api에게 요청을 보내는데 400 bad request가 발생하였습니다. 처음에는 json 형태에서 
            \'를 인식 못하는 것으로 판단하여 replace 함수를 통해 \' 를 \" 로 변환하여 넘겨 주었습니다. 
            그럼에도 불구하고 같은 문제가 발생하였고 다시한번 glanceclient 코드를 확인해보니 
            json 함수를 이용하여 값을 json형태로 직접 변경해서 넣어줘야 한다는 점을 알게 되었습니다. 
            따라서, json 함수를 import 하여 json.loads 로 input의 값을 json형태로 변경해주었습니다. 

    4. 완성된 코드
        
        이와 같은 과정을 걸쳐서 코드를 만들어 냈으며 실제 값을 출력하는지도 확인해보았습니다.

        .. code-block:: python

         class CreateTask(command.ShowOne):
            _description = _("Create a new task from attributes")

            def get_parser(self, prog_name):
             parser = super(CreateTask, self).get_parser(prog_name)
             parser.add_argument(
                    "--type",
                    metavar="<type>",
                    help=_("Type of Task. Please refer to Glance schema or "
                        "documentation to see which tasks are supported."),
                )
                parser.add_argument(
                    "--input",
                   metavar="<STRING>",
                   default='{}',
                   help=_("Parameters of the task to be launched."),
                 )
             return parser

            def take_action(self, parsed_args):
                kwargs = {}
                info = {}
             image_client = self.app.client_manager.image
             copy_attrs = ('type', 'input')
                for attr in copy_attrs:
                    if attr in parsed_args:
                       val = getattr(parsed_args, attr, None)
                       if val:
                           # Only include a value in kwargs for attributes that
                           # are actually present on the command line
                           val = val.replace("'", "\"")  # To use json.loads
                           if attr == 'input':
                             val = json.loads(val)
                             kwargs[attr] = val
                         else:
                             kwargs[attr] = val

                task = image_client.create_task(**kwargs)
                if not info:
                    info = _format_image(task)
                return zip(*sorted(info.items()))


    .. image:: images/create_task_2.png
        :height: 300
        :width: 700
        
    .. image:: images/create_task_3.png
        :height: 300
        :width: 700
 

    * 다음과 같이 실제로 동작하게 되었으며, wip로 코드를 올렸을 때, zuul 을 통과했습니다! 
      (test case 구현도 해야해서 wip로 우선 했습니다.)
      이를 바탕으로 test case 작성을 하게 되었습니다.
      이렇게 커맨드를 추가하면서 openstackclient 에서 어떤식으로 api를 호출하게 되는지
      알게 되었으며 직접 api를 호출하는 커맨드를 짤 수 있게 되었습니다. 저는 코딩을 잘하는
      편이 아니라고 느꼈는데 꾸준한 디버그와 멘토,멘티님들과의 소통을 통해서 큰 프로젝트의 커맨드를
      만들 수 있었습니다. 이걸 보고 계시는 분들도 포기하지 마시고 화이팅 하길 바랍니다!!

