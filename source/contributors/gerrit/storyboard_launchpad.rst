:orphan:

OpenStack Storyboard & LaunchPad 알아보기
=======================================================================

1. Storyboard란?
------------------------------------------------

OpenStack이라는 광활한 오픈소스 프로젝트에서 발생하는 여러가지 버그, 기능 개선, 해결된 사항들을 Github에서처럼 Issue를 만들어 프로젝트에 참여하고 있는 사람들에게 안내를 해야하는 상황은 필수일 것입니다.

이 때 OpenStack에서는 Storyboard라는 툴을 사용하고 있으며 아래 이미지에서 설명하는 것처럼 버그 리포트와 가능 개선에 대한 스토리를 기록하고 공유하는 목적으로 사용됩니다.

.. image:: https://user-images.githubusercontent.com/40455392/137765076-68113a5f-9f74-42a6-b066-76d7ca6573b2.png
   :width: 1000px

이 스토리는 하나의 Task 단위로 관리되며, "OpenStack Storyboard 사용 방법 안내" 글에서 보면, 추후에 커밋 작업을 진행할 때, Gerrit에서 Task ID를 기록해 이슈 트래킹을 가능하게 지원합니다.

각각의 스토리는 OpenStack의 컴포넌트 저장소 별로 관리되며, 이후에 Sandbox라는 연습용 레포에서 스토리보드의 이슈를 생성하고, Gerrit을 사용해 코드 리뷰하는 과정을 직접 경험해 보게 될 것입니다.

2. LaunchPad란?
------------------------------------------------

Launchpad는 StoryBoard 으로 이동하기 시작한 이래 OpenStack 커뮤니티에서 버그 추적을 하는 기존 방식으로, 마찬가지로 Storyboard 이전의 버그 리포트와 기능 개선 등을 관리하는 툴이라 보면 됩니다.

.. image:: https://user-images.githubusercontent.com/40455392/137766262-36c12374-3bf6-4d98-868f-153bc0c0fa68.png
   :width: 1000px

3.Reference
------------------------------------------------

- `Storyboard Main Website <https://storyboard.openstack.org/>`_

- `LaunchPad Website <https://launchpad.net/openstack>`_

