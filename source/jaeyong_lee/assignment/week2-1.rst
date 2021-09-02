CLI와 친해지기
==========================================================

1. cirros image로 인스턴스 생성을 cli로 해보기
___________________________________________________________
 1. $ openstack create -image=cirros-0.5.2-x86_64-disk --flavor=m1.tiny --network=public test_instance   
    
    * 인스턴스 생성
        .. image:: ../images/week2-1_0.png
            :height: 500
            :width: 700
            :scale: 100
            :alt: cirros image 생성




2. ubuntu 이미지를 받고, root password를 설정한 다음 cli로 이미지 등록한 후 인스턴스 생성하고 접속까지 하기
___________________________________________________________________________________________________________________

 1. ubuntu 이미지 받기 및 root password 설정
     
     .. code-block::

      $ mkdir /images  
      $ cd /images 
      $ wget https://cloud-images.ubuntu.com/focal/current/focal-server-cloudimg-amd64.img 
      $ sudo apt-get install libguestfs-tools 
      $ sudo virt-customize -a bionic-seerver-cloudimg-amd64.img -root-password password:0000 //0000에 원하는 비밀번호 

     

 2. 이미지 등록 후 인스턴스 생성
     
     .. code-block::

      $ source openrc admin admin    
      $ openstack images create "ubuntu" --file images/focal-server-cloudimg-amd64.img --disk-format raw --container-format bare --public
      $ source openrc admin demo
      $ openstack create -image=ubuntu --flavor=m1.small --network=private cli_instance
     

     * 이미지 등록
        .. image:: ../images/week2-1_1.PNG
            :height: 500
            :width: 1000
            :scale: 100
            :alt: 이미지 등록
    
     
     * 인스턴스 생성
        .. image:: ../images/week2-1_4.png
            :height: 500
            :width: 700
            :scale: 100
            :alt: 이미지 설명

3. cli로 floating ip 생성 후 인스턴스에 할당 / 해제 해보기
_________________________________________________________________________________
 1. floating ip 생성
     .. code-block::
     
      $ openstack floating ip create public 

    .. image:: ../images/week2-1_6.png
        :height: 500
        :width: 700
        :scale: 100
        :alt: floating ip create


 2. floating ip 할당
    
     .. code-block::

      $ openstack port list  // 연결할 port id 복사
      $ openstack floating ip set --port {연결할 포트} --fixed-ip-address {private ip} {floating ip}

     * floating ip 할당

        .. image:: ../images/week2-1_7.png
            :height: 100
            :width: 1200
            :scale: 70
            :alt: floating ip set

     *  할당 결과

        .. image:: ../images/week2-1_8.png
         :height: 300
         :width: 1000
         :scale: 70
         :alt: floating ip set result


 3. floating ip 해제 

     .. code-block::

      $ openstack ip unset —port {할당한 floating ip}
     
     * 할당 해제 및 결과
        .. image:: ../images/week2-1_10.png
            :height: 200
            :width: 700
            :scale: 100
            :alt: floating ip unset
