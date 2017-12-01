# Splunk UniversalForwarder자동설치 (Ansible오픈소스 이용)
*****************************************************************

목 차

I.	ANSIBLE이란	
1.	개념	
2.	용어정리	

II.	설치 및 구성	
1.	구성도	
2.	ANSIBLE설치 	
3.	PLAYBOOK를 이용한 자동설치	

----

## I. ANSIBLE이란	
## 1. 개념 

- 리모트 서버에 접속해서 환경설정, 배포등을 가능하게 하는 오픈소스
<br>참고 :http://docs.ansible.com/ansible/latest/intro.html)

## 2. 용어정리 (* Facebook Ansible Korea User Group 참고) 

1)inventory : 작업의 대상이 되는 호스트들을 의미. 주로/etc/ansible/hosts 파일에 기록하며 ip address 또는 hostname으로 정의할 수 있음 .또한 이 파일에서 각 호스트들의 그룹을 지정할 수 있기 때문에 다수의 서버들을 한번에 관리할 때 매우 유리

2)Module : 작업도구로써 ansible이 실행시키는 모든 job은 이 module들을 기반으로 생성됨 ansible-doc \<module\_nanme\> 명령어를 통해 현재 버전의 ansible이 어떤 모듈을 지원하고 어떻게 사용해야하는지 확인가능

3)Playbook : 한마디로 작업계획서 라고 할 수 있음. 작업대상의 호스트들에 대해 어떤 작업을 할것인지(파일 업로드, 복사, 등등) yaml문법 형태로 정의해놓은 것 yaml형식은 key-value방식인데 python에서와 같이 indent에 주의해야 함 즉 줄간격을 잘 맞춰야 함


## II.	설치 및 구성	
## 1.	구성도	

1)2대의 VM에 각각 cent os 7 linux를 설치하여 1대에는 Ansible을 설치하고 나머지 1대는 splunk Universal forwader를 자동으로 설치할 예정이다.

2)Ansible이 설치되면 ansible-test라는 Host에서 실행한 명령어 및 command가 host: splunk\_cent7에 전달된다   

![구성도](https://github.com/euimokna/ansible/blob/master/ansible_splunk.png)

## 2.	ANSIBLE설치 	

- 설치 및 구성

(1) anslbie 설치 :  /etc/ansible경로에 ansible이 설치됨 아래와 같은 하위 폴더가 생김 

``` 
[root@ansible-test ~]# yum install ansible 

drwxr-xr-x. 2 root root     6  6월  2 06:49 roles
-rw-r--r--. 1 root root 18066  6월  2 06:49 ansible.cfg
-rw-r--r--. 1 root root  1097  9월 13 23:28 hosts
```

(2) ssh키 등록

```
 키생성 :ssh-keygen -t rsa                                          
                                                                   
 키복사 :  ssh-copy-id -i \~/.ssh/id\_rsa.pub \[user\]@\[host\]     
                                                                   
 예시)ssh-copy-id -i /root/.ssh/id\_rsa.pub root@192.168.244.20    
```

(3)	리모트 서버 IP등록 

```
vi /etc/ansible/hosts 

[splunk] 
192.168.244.20 
```

- Ansible

(1)/etc/ansible/hosts 등록된 서버에 ansible명령어로 ping을 날려본다. 

```
실행 : [root@ansible-test ansible]# ansible -m ping all

결과 
192.168.244.20 | SUCCESS => {
"changed": false,
"ping": "pong"
}
```
(2) Ad-Hoc command를 통해  파일 upload를 해본다. (참고 http://docs.ansible.com/ansible/latest/intro_adhoc.html) 

```
예시 : ansible </etc/ansible/hosts등록된 명> -m <모듈명> "src=<로컬파일경로> dest=<리모트파일경로>" -u root

실행: ansible splunk -m copy -a "src=/root/test_file.tar dest=/opt/test_file.tar" -u root

실행결과
[root@ansible-test ansible]# ansible splunk -m copy -a "src=/root/test_file.tar dest=/opt/test_file.tar" -u root 
192.168.244.20 | SUCCESS => {
 "changed": true,
 "checksum": "09d1a3c5e4b6bfb9a03309e7e6ae9120355ffede",
"dest": "/opt/test_file.tar",
"gid": 0,
"group": "root"
 "md5sum": "f51c712f04c4eeefe2f42d9679364823",
"mode": "0644",
 "owner": "root",
 "secontext": "system_u:object_r:usr_t:s0",
"size": 10240,
"src": "/root/.ansible/tmp/ansible-tmp-1505452178.83-191758065171232/source",
"state": "file"
 "uid": 0
}

대상(splunk universal forwarder서버) 확인 
[root@splunk_cent7 ~]# cd /opt
[root@splunk_cent7 opt]# ls -rtl
-rw-r--r--. 1 root root       10240  9월 15 14:09 test_file.tar
```

## 3.Playbook를 이용한 자동설치
1)Ad-Hoc검색은 매번 command 명령어를 날려야 하기 때문에 한번에 실행할 수 있는 playbook을 만든다. 앞서 add-hoc command를  yaml문법형태로 정의하면 된다. (참고 http://docs.ansible.com/ansible/latest/playbooks.html) 

2)/etc/ansible에 playbooks라는 디렉토리를 만들고 splunk_install.yml이라는 파일을 만들어서 아래의 내용을 넣는다.
```
---
- hosts : splunk
  tasks :
  - name : file transfer
    copy :
     src : "/root/splunkforwarder-6.6.3-e21ee54bc796-Linux-x86_64.tgz"
     dest : "/opt/splunkforwarder-6.6.3-e21ee54bc796-Linux-x86_64.tgz"
     mode : 0700
  - name : check splunkforwarder if exist remove
    stat : path=/opt/splunkforwarder
    register : result
    ignore_errors : True
  - file :
     path : /opt/splunkforwarder
     state : absent
    when : result|succeeded
  - name : unarchive splulnk files
    unarchive :
     src : "/opt/splunkforwarder-6.6.3-e21ee54bc796-Linux-x86_64.tgz"
     dest : "/opt"
     remote_src: True
  - file :
     path : /opt/splunkforwarder
     state : directory
     owner : root
     group : root
  - lineinfile:
     path : /etc/security/limits.conf
     regexp: '^sEnd of file'
     line: "{{item.domain}}           {{item.type}}        {{item.item}}        {{item.value}}"
    with_items:
       - {domain: '*', type: 'hard', item: 'nproc', value: '10240'}
       - {domain: '*', type: 'soft', item: 'nproc', value: '10240'}
       - {domain: '*', type: 'hard', item: 'nofile', value: '10240'}
       - {domain: '*', type: 'soft', item: 'nofile', value: '10240'}     
  - name: auto start register
    command : /opt/splunkforwarder/bin/splunk enable boot-start -user root --accept-license
  - name: splunk start
    command : /opt/splunkforwarder/bin/splunk start
```
3)[root@ansible-test playbooks]# Ansible-playbook splunk_install.yml 명령어 수행 결과 

```
root@ansible-test playbooks]# ansible-playbook splunk_install.yml

PLAY [splunk] ***************************************************************************************************

TASK [Gathering Facts] ******************************************************************************************
ok: [192.168.244.20]

TASK [file transfer] ********************************************************************************************
ok: [192.168.244.20]

TASK [check splunkforwarder if exist remove] ********************************************************************
ok: [192.168.244.20]

TASK [file] *****************************************************************************************************
changed: [192.168.244.20]

TASK [unarchive splulnk files] **********************************************************************************
changed: [192.168.244.20]

TASK [file] *****************************************************************************************************
changed: [192.168.244.20]

TASK [lineinfile] ***********************************************************************************************
changed: [192.168.244.20] => (item={u'item': u'nproc', u'domain': u'*', u'type': u'hard', u'value': u'10240'})
changed: [192.168.244.20] => (item={u'item': u'nproc', u'domain': u'*', u'type': u'soft', u'value': u'10240'})
changed: [192.168.244.20] => (item={u'item': u'nofile', u'domain': u'*', u'type': u'hard', u'value': u'10240'})
changed: [192.168.244.20] => (item={u'item': u'nofile', u'domain': u'*', u'type': u'soft', u'value': u'10240'})

TASK [auto start register] **************************************************************************************
changed: [192.168.244.20]

TASK [splunk start] *********************************************************************************************
changed: [192.168.244.20]

PLAY RECAP ******************************************************************************************************
192.168.244.20             : ok=9    changed=6    unreachable=0    failed=0
```

4)대상서버(splunk Universl forwarder)에 splunk설치 및 기동 학인 
```
[root@splunk_cent7 opt]# ls -rtl
-rw-r--r--. 1 root root       10240  9월 15 14:09 test_file.tar
drwxr-xr-x. 9 root root         231  9월 15 17:39 splunkforwarder

[root@splunk_cent7 opt]# ps -ef | grep splunkd
root      25809      1  0 17:39 ?        00:00:00 splunkd -p 8089 start
root      25814  25809  0 17:39 ?        00:00:00 [splunkd pid=25809] splunkd -p 8089 start [process-runner]
root      25865   3744  0 17:44 pts/0    00:00:00 grep --color=auto splunkd

[root@splunk_cent7 opt]# cat /etc/security/limits.conf
……
……
#<domain>      <type>  <item>         <value>
#

#*               soft    core            0
#*               hard    rss             10000
#@student        hard    nproc           20
#@faculty        soft    nproc           20
#@faculty        hard    nproc           50
#ftp             hard    nproc           0
#@student        -       maxlogins       4

# End of file
*           hard        nproc        10240
*           soft        nproc        10240
*           hard        nofile        10240
*           soft        nofile        10240
```
