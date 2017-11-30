# Splunk UniversalForwarder자동설치(Ansible오픈소스 이용)
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

1)  inventory : 작업의 대상이 되는 호스트들을 의미. 주로
    /etc/ansible/hosts 파일에 기록하며 ip address 또는 hostname으로
    정의할 수 있음 .또한 이 파일에서 각 호스트들의 그룹을 지정할 수 있기
    때문에 다수의 서버들을 한번에 관리할 때 매우 유리

2)  Module : 작업도구로써 ansible이 실행시키는 모든 job은 이 module들을
    기반으로 생성됨 ansible-doc \<module\_nanme\> 명령어를 통해 현재
    버전의 ansible이 어떤 모듈을 지원하고 어떻게 사용해야하는지 확인가능

3)  Playbook : 한마디로 작업계획서 라고 할 수 있음. 작업대상의
    호스트들에 대해 어떤 작업을 할것인지(파일 업로드, 복사, 등등)
    yaml문법 형태로 정의해놓은 것 yaml형식은 key-value방식인데
    python에서와 같이 indent에 주의해야 함 즉 줄간격을 잘 맞춰야 함



## II.	설치 및 구성	
## 1.	구성도	

1)  2대의 VM에 각각 cent os 7 linux를 설치하여 1대에는 Ansible을
    설치하고 나머지 1대는 splunk Universal forwader를 자동으로 설치할
    예정이다.

2)  Ansible이 설치되면 ansible-test라는 Host에서 실행한 명령어 및
    command가 host: splunk\_cent7에 전달된다.
    
    

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
(1)	/etc/ansible/hosts 등록된 서버에 ansible명령어로 ping을 날려본다. 

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
