# NFS, SMB, iSCSI

# NFS, SMB, ISCSI Usage

## Setup

Hypervisor: Virutual Box 6.1

OS: CentOS7.9.minimal

Network: NAT, HostOnly Adapter

## NFS

### Server

1. 패키지 설치

    ```bash
    $ yum -y install nfs-utils
    ```

2. nfs 설정

    ```bash
    $ vi /etc/exports
    <SHARE FOLDER PATH> <ALLOW IP PREFIX>(OPTIONS)
    /nfs    192.168.122.0/24(rw,sync,sec=sys,no_root_squash)
    # no_root_squash 해줘야 클라이언트에게 root권한 물어보지 않음
    ```

3. 설정 확인

    ```bash
    $ exportfs -r # export 테이블 다시 불러오기
    $ exportfs -v # 파일 내용 확인
    ```

4. 서비스 시작 및 방화벽 등록

    ```bash
    $ systemctl enable --now nfs
    $ firewall-cmd --permanent --add-service=nfs

    # rpc-bind, mountd 를 방화벽에 서비스 등록 해주면
    # 클라이언트에서 showmount 로 서버의 공유폴더 정보 확인 가능
    $ firewall-cmd --permanent --add-service=rpc-bind
    $ firewall-cmd --permanent --add-service=mountd
    ```

### Client

1. 패키지 설치

    ```bash
    $ yum -y install nfs-utils
    ```

2. 서버 검색 및 마운트
    1. showmount

        ```bash
        $ showmount -e <SERVER IP>
        ```

    2. 수동 마운트

        ```bash
        # 1. command
        # $ mount -t nfs <SERVER IP>:<PATH> <MOUNTPOINT>
        $ mount -t nfs 192.168.123.20:/nfs /mnt/nfs-client 
        # 2. fstab
        $ vi /etc/fstab
        # SERVERIP:PATH MOUNTPOINT FSTYPE OPTIONS 0 0
        192.168.123.20:/nfs /mnt/nfs-client nfs defaults,sec=sys 0 0
        $ mount -a
        ```

3. 자동 마운트(autofs)

    1. 패키지 설치

    ```bash
    $ yum -y install autofs
    ```

    2. 직접 자동 마운트

    1. 마운트 포인트 폴더 생성

        ```bash
        $ mkdir /mnt/public
        ```

    2. 마스터 맵 파일 작성

        ```bash
        $ vi /etc/auto.master.d/<NAME>.autofs
        /- /etc/auto.<MAPFILENAME>
        ```

    3. 직접 맵 파일 작성

        ```bash
        $ vi /etc/auto.<MAPFILENAME>
        <MOUNTPOINT> -<OPTIONS> <SERVERIP>:<PATH>
        #/mnt/public     -rw,sync,sec=sys    192.168.122.200:/shares/public
        ```

    4. 서비스 재시작

        ```bash
        $ systemctl restart autofs
        ```

    3. 간접 자동 마운트

    1. 마스터 맵 작성

    ```bash
    #<MOUNTPOINT> 는 사전에 생성되선 안됨, autofs가 자동으로 생성함
    $ vi /etc/auto.master.d/<NAME>.autofs
    <MOUNTPOINT> /etc/auto.<MAPFILENAME>
    ```

    2. 간접맵 파일 작성

    ```bash
    1. 1:1 방식
    $ vi /etc/auto.<MAPFILENAME>
    <SUB MOUNT POINT> -<OPTIONS> <SERVERIP>:<PATH>
    # public     -rw,sync,sec=sys    192.168.122.200:/shares/public

    2. 1:n 방식
    $ vi /etc/auto.<MAPFILENAME>
    * -<OPTIONS> <SERVERIP>:<PARENT_PATH>/&
    # *     -rw,sync,sec=sys    192.168.122.200:/shares/&
    ```

    3. 서비스 재시작

    ```bash
    $ systemctl restart autofs
    ```

---

## SMB

### Server

1. Install Package

    ```bash
    $ yum -y install samba samba-client
    ```

2. 공유 디렉토리 생성, 사용자 생성

    ```bash
    $ mkdir -p /share/samba
    $ useradd -s /sbin/nologin smbuser1
    ```

3. SELinux context 설정 or Off SELinux

    ```bash
    $ semanage fcontext -a -t samba_share_t '/share/samba(/.*)?'
    $ restorecon -RFv /share/samba/
    or
    $ setenforce 0
    ```

4. smb 설정파일 편집

    ```bash
    $ vi /etc/samba/smb.conf
    ...
    [test] #SECTION-NAME
            comment = samba test
            path = /share/samba
            valid users = smbuser1
            write users = smbuser1
            browseable = no
    ...
    $ testparm # 설정확인
    ```

5. 서비스 시작 및 방화벽 설정

    ```bash
    $ systemctl enable --now smb nmb
    $ firewall-cmd --permanent --add-service=samba
    $ firewall-cmd --reload
    ```

6. SMB 사용자 등록 및 확인

    ```bash
    $ smbpasswd -a smbuser1
    $ file /var/lib/samba/private/passdb.tdb # 여기에 저장됨
    $ pdbedit --list # 이 명령어로 smb에 등록된 사용자 확인
    ```

### Client

1. 패키지 설치

    ```bash
    $ yum -y install cifs-utils # smb가 사용하는 파일 시스템을 지원하는 패키지
    $ yum -y install samba-client # smb 클라이언트 도구 서버에서 제공하는 공유 섹션 탐색
    ```

2. [Optional]서버의 smb.conf에서 browseable 이 yes일 때 탐색 가능

    ```bash
    $ smbclient -L SERVER-ADDRESS -u SMB-USERNAME
    ```

3. 수동 마운트
    1. 비밀번호 직접 입력 방법

        ```bash
        $ mount -o username=smbuser1 //SERVER-IP/SECTION-NAME MOUNTPOINT
        ```

    2. 비밀번호 credential 파일로 관리하는 방법

        ```bash
        $ vi /cred # CREDENTIALS FILE PATH
        username=SERVER에 등록된 SMBUSER-NAME
        password=PASSWORD

        1. 직접 사용
        $ sudo mount -o credentials=CREDENTIAL-FILE-PATH //SERVER-IP/SECTION-NAME MOUNTPOINT

        2. 마운트 포인트 등록
        $ vi /etcfstab
        //SERVER-IP/SECTION-NAME MOUNTPOINT cifs credentials=CREDENTIAL-FILE-PATH 0 0
        $ mount -a
        ```

4. 자동 마운트
    1. 직접맵

        ```bash
        $ vi /etc/auto.master.d/<NAME>.autofs # 마스터 맵 파일 생성
        /- /etc/auto.<MAP-FILE-NAME>

        $ vi /etc/auto.<MAP-FILE-NAME> # 직접 맵 파일 생성
        MOUNTPOINT -fstype=cifs,credentials=/cred ://SERVER-IP/SECTION-NAME

        $ systemctl restart autofs
        ```

    2. 간접맵

        마스터 맵의 마운트 포인트 '/share'와  맵파일의 마운트 포인트 'test'는 자동 생성됨

        해당 경로로 이동하면 마운트가 자동으로 이루어짐( /samba/test

        ```bash
        $ vi /etc/auto.master.d/<NAME>.autofs # 마스터 맵 파일 생성
        /samba /etc/auto.<MAP-FILE-NAME>

        $ vi /etc/auto.<MAP-FILE-NAME>
        test -fstype=cifs,credentials=/cred ://SERVER-IP/SECTION-NAME

        $ systemctl restart autofs
        ```

---

## iSCSI

### Server

1. 제공할 디스크 파티셔닝

    ```bash
    $ fdisk /dev/sdb
    sdb1 생성

    $ pvcreate dev/sdb1
    $ vgcreate vg1 /dev/sdb1 [-s 로 PE 크기 지정 가능)
    $ lvcreate vg1 -n lv1 -L 5G [-l로 pe 개수 지정 가능]
    # 파티셔닝으로 탄생한 경로: /dev/vg1/lv1
    ```

2. Install targetcli & ISCSI 구성

    ```bash
    $ yum -y install targetcli
    $ targetcli
    # 1. backstore에서 block 생성
    /backstores/block> create name=<NAME> dev=<파티션장치명>
    # 2. ISCSI 연결 설정
    # IQN 주소(호스트) 설정
     /iscsi> create wwn=iqn.<생성날짜 2020-06>.<호스트네임(.com)>:<TAG(server)>
    # ACL(클라이언트) 설정
     /iscsi/iqn.<생성날짜 2020-06>.<호스트네임(.com)>:<TAG(server)>/tpg1/acls> create wwn=iqn.<날짜>.<호스트(.com)>:client
    # LUN 설정 (lun 번호 지정 안하면 자동생성)
    /iscsi/iqn.<생성날짜 2020-06>.<호스트네임(.com)>:<TAG(server)/tpg1/luns> create storage_object=<backstore에서 생성한 block>
    # portal은 자동으로 0.0.0.0:3260 등록
    ```

3. 서비스 재시작 및 방화벽 등록

    ```bash
    $ systemctl restart target
    $ firewall-cmd --permanent --add-port=3260/tcp
    $ firewall-cmd --reload
    ```

### Client

1. 패키지 설치

    ```bash
    $ yum -y install iscsi-initiator-utils
    ```

2. IQN 설정

    ```bash
    $ vi /etc/iscsi/initiatorname.iscsi
    InitiatorName=iqn.2020-12.hjserver.com:client
    ```

3. 서비스 활성화

    ```bash
    $ systemctl enable --now iscsid
    ```

4. 연결
    1. 검색

        ```bash
        $ iscsiadm -m discovery 0t st -p <TARGETSERVERIP>
        ```

    2. 로그인

        ```bash
        # -l 옵션이 로그인, -m은 node로 해야 연결을 시도
        $ iscsiadm -m node -T iqn.2020-12.hjserver.com:server -l
        ```

    3. 세션 확인

        ```bash
        $ iscsiadm -m session
        ```

    4. 연결 해제

        ```bash
        # 로그아웃(연결은 해제 되지만 정보는 남아있어서 이후 검색없이 다시 로그인 가능)
        $ iscsiadm -m node -T iqn.2020-12.hjserver.com:server -u
        # 완전한 연결 해제(-u로 로그아웃 후 -o 로 연결 정보를 삭제
        # 이후 다시 연결을 할 경우엔 검색 부터 해야 한다.
        $ iscsiadm -m node -T iqn.2020-12.hjserver.com:server -o
        ```