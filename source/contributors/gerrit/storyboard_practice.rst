:orphan:

OpenStack Sandbox 저장소를 이용해서 2인 1조로 코드 리뷰 연습하기
=======================================================================

1. git review를 이용한 커밋 메세지 작성 방법 및 푸시 방법 안내
------------------------------------------------------------------------------------------------
- Sandbox를 사용하고 싶다면 프로젝트를 클론을 해줘야 합니다 (하단 Sandbox Repo reference 참고).

- 위의 링크를 클론해서 당신의 프로젝트 폴더에 넣고 해당 폴더의 경로로 들어가서 아무 파일이나 생성합니다.

- 저는 brain.py라고 파일을 생성후 print 문을 작성했습니다.

.. image:: https://miro.medium.com/max/1400/1*zdjP3J74Y7RI9cl1LfJRGQ.png
   :width: 1000px

- 파일들 명칭만 봐도 알겠지만 모두 테스틀 위해 대충 만든 레포라는 것을 볼 수 있습니다.

- 그리고 아래 명령어를 작성하면, 커밋 메세지 작업 규칙이 나와 있습니다.

.. code-block:: console

    > git add brain.py
    > git commit

- 작업 규칙은 제목 및 본문은 한 줄에 79자 내외, 그리고 storyboard & task ID를 하단에 작성해주는 것입니다.

.. image:: https://miro.medium.com/max/555/1*oQJliQaHshpeduLJTiBW7Q.png
   :width: 1000px

- 위의 이미지처럼 규칙을 지켜서 작성 후에 :wq를 입력하고 저장해줍니다.

- 그리고 git push origin <원본 레포>가 아닙니다.

- git review를 입력해주면 코드가 자동 push 되는 것을 볼 수 있습니다.

.. image:: https://miro.medium.com/max/700/1*C82ZJ0mhE0_Unq9cMUsrMg.png
   :width: 1000px

2. Gerrit을 사용한 코드 리뷰 남기기 및 수정 코드 커밋
----------------------------------------------------------------------------------

- 아래 이미지는 위의 링크를 따라 접속하면 볼 수 있는 Gerrit 코드 리뷰 페이지 입니다.

.. image:: https://miro.medium.com/max/1400/1*URKkXK0JFdGmK53PgwbSzg.png
   :width: 1000px


- 그럼 위의 페이지에서 제가 수정한 파일을 들어갈 수 있고, 거기서 키보드의 c를 누르면 누군가의 코드에 대해 리뷰를 남길 수 있습니다.

.. image:: https://miro.medium.com/max/700/1*hoRyr1U_J6B_zqrNMz9wtw.png
   :width: 1000px

- (참고로 본인이 커밋한 코드에 대해서는 셀프 코드 리뷰가 불가능합니다.)

- 아마 누군가 리뷰를 남겨줬다면 Code-Review 점수가 올라가 있을 것입니다.

.. image:: https://miro.medium.com/max/544/1*0NqcMl895GW8GOi0MnURpw.png
   :width: 1000px

- 처음 올린 코드 커밋에 대해 확인이 필요하면 리뷰를 남기고 0점을 주고 (아직 완성 못했으므로) 수정했다면 리뷰 점수를 일반 사용자는 2점까지 줄 수 있습니다.

.. image:: https://miro.medium.com/max/536/1*qeL-2ZW93davU_ScaMU0mg.png
   :width: 1000px


- 그리고 최종 merge를 하기 위해서는 workflow라는 점수가 있는데 이는 프로젝트를 매니징하거나 관리하는 분들이 줄 수 있기 때문에 일반 개발자가 부여할 수 없는 점수라고 보면 됩니다.

.. image:: https://miro.medium.com/max/700/1*vHnJguEsNDKIv8lijKCLxA.png
   :width: 1000px

- 그리고 -2, -1 점도 있는데 이건 코드에 심각한 결함이 있거나(-1점) 관리자가 해당 개발을 거부하는 경우 (-2점에 해당하며 코드를 잘못짜서 주는 점수는 아니라고 한다)라고 보면 됩니다.

- (당연히 위의 경우 merge는 될 수 없습니다.)

- 누군가 코드 리뷰를 남겨주면, 그 코드 리뷰에 따라 내가 작업한 코드를 수정해주고, 다시

.. code-block:: console

    > git add brain.py
    > git commit --ammend

- 를 작성해서 기존 커밋을 수정해서 다시 git review를 해줍니다.

- (기존 파일 수정본에 대해서는 새롭게 커밋하는 게 아니라서 해당 부분을 조심하셔야 합니다!)

3. Gerrit에서 코드가 Merge되는 방법
------------------------------------------------

.. image:: https://miro.medium.com/max/1400/1*BlpWVBfgEjbuK9mG68QXZg.png
   :width: 1000px

- 이렇게 되면 여기서 확대해서 볼 부분이 있는데

.. image:: https://miro.medium.com/max/648/1*idw5lTwc43BO18LmruZ4tg.png
   :width: 1000px

- 위의 코드 리뷰 화면 상단에 Base라고 되어 있는 부분을 Patchset1 로 변경하고 코드 수정을 완료하면 코드 리뷰 남겨준 사람에게 코드를 수정했다고 언급해주고 Done을 클릭합니다.

- (본인이이 커밋한 사람이 아니라 리뷰를 남긴 사람일 때 해당)

.. image:: https://miro.medium.com/max/700/1*Lxko2Bc0rolgsKLm7VTgyw.png
   :width: 1000px

- 그럼 리뷰를 남긴 사람의 글 오른편에 patchset2라고 되어 있는 것을 볼 수 있고 (처음 커밋에 리뷰를 남기면 patchset1이라 되어 있다) 최종 승인자가 workflow 점수를 주면 해당 코드가 merge되어 gerrit과 storyboard 모두 close 됩니다.


4. 에러 처리
------------------------------------------------

4–1) Remote rejected Error (git review 명령어 진행 시)
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

.. image:: https://miro.medium.com/max/1400/1*EXRhwaIlcurUU6ISgmc20g.png
   :width: 1000px

- 팀원이 신규 커밋을 올려서 merge가 된 경우가 있었고, 새로운 커밋 히스토리가 생겼으므로 git pull을 해서 당신의 로컬 레포를 최신화시켜 줍니다.

- 만약 당신이 멋모르고 커밋을 먼저 했다면 (== 나) git reset — hard HEAD~1 라고 해서 당신 커밋기록을 지우고 다시 git pull을 해주면 됩니다.

4–2) Base → Patchset1로 변경 안하고 최종 커밋 커멘트 작성시 해결
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

- 해당 커멘트를 수정하고, Base → Patchset1로 변경 후, Code Review & Workflow 점수를 부여합니다.

- (물론 실제 의미있는 코드에 대한 커밋이라면 Workflow 점수는 부여할 수 없습니다. 다만 현재는 테스트 상이니까 가능한 일입니다!)

- 그러면 Zuul이 알아서 CI를 돌면서 Storyboard & Gerrit 모두 자동 merge를 진행하게 됩니다.

5.Reference
------------------------------------------------

- `Sandbox Repo <https://opendev.org/opendev/sandbox>`_

- `OpenStack Code Review 방법 <https://docs.openstack.org/project-team-guide/review-the-openstack-way.html>`_