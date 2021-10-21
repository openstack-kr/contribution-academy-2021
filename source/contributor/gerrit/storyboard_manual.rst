:orphan:

OpenStack Storyboard 사용 방법 안내
=======================================================================

1. Storyboard 사용하기
------------------------------------------------

- 아마 "OpenStack Gerrit 계정 생성 및 커밋 환경 설정하기" 내용을 읽고 넘어오셨다면, StoryBoard는 당연히 소셜 로그인 연동이 될테니 가입 절차는 생략하겠습니다.

- storyboard를 들어오면 프로젝트 검색 창에 sandbox라는 것이 있습니다 (하단 reference 참고).

- Storyboard는 전 세계 모든 OpenStack 사용자들이 OpenStack을 사용하면서 발생한 버그 등의 이슈를 올려놓는 곳인데, 그 중에 Sandbox라는 프로젝트는 Gerrit을 이용한 코드리뷰 테스트를 작업하기 위해 생성한 레포입니다.

- (그래서 모래 상자라고 해서 엉망으로 사용해도 문제가 안되니 프로젝트 이름을 지었나 유추해봅니다.)

.. image:: https://miro.medium.com/max/700/1*DnBa5gLkYBizSlfzyt02oA.png
   :width: 1000px

- 샌드박스 스토리 보드를 확인하고, "add stroy"를 클릭합니다.

.. image:: https://miro.medium.com/max/700/1*hD8brKFKja1GeLioYbcRpg.png
   :width: 1000px

- 당신이 sandbox 계정을 생성했다면 이제 sandbox에 첫번째 이슈를 생성해줍니다.

- Add Story를 눌러서 이슈 제목과 내용을 마음껏 작성하고, Save Changes를 클릭해주면 첫번째 스토리 보드가 생성되는 것을 볼 수 있습니다.

.. image:: https://miro.medium.com/max/700/1*UbDL-JSmH_fcCYJE5XvGXQ.png
   :width: 1000px

- 그럼 이슈가 정상적으로 등록되고 여기서 2가지가 확인 가능합니다.

    - Story ID : URL의 뒷부분 번호 확인
    - Task ID : task 하단의 5자리 번호 확인

- 이 2가지를 이제 커밋할 때 반드시 기록해야합니다.

- Storyboard에 이슈를 생성하는 방법에 대해 알아봤으니, 이제 다음 글에서 2인 1조로 코드 리뷰를 하는 방법에 대해 실습해보겠습니다.

2.Reference
------------------------------------------------

- `Storyboard Sandbox <https://storyboard.openstack.org/#!/project/opendev/sandbox>`_

- `Sandbox Repo <https://opendev.org/opendev/sandbox>`_

- `OpenStack Code Review 방법 <https://docs.openstack.org/project-team-guide/review-the-openstack-way.html>`_

- `스토리보드 공식 설명 문서 <https://docs.openstack.org/infra/storyboard/>`_