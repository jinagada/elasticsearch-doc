# Elastic Stack 설치 가이드
- 이 문서에서는 Elasticsearch 6.4.0 기준으로 설치가 진행 되지만, 6.5.4 버전도 동일하게 설치 할 수 있음

# 테스트 환경

- VirtualBox : 5.2.16 r123759(Qt5.6.2)
  - CPU : 2 cpu
  - MEM : 4G
  - HDD : 100G
  - 머신수 : Elasticsearch X 3, Kibana X 1, Logstash X 1, Beats X 1
- 각 가상머신 별 IP / HOST 명
  - 192.168.56.101 elastic-01
  - 192.168.56.102 elastic-02
  - 192.168.56.103 elastic-03
  - 192.168.56.104 kibana_04
  - 192.168.56.105 logstash_05
  - 192.168.56.106 beats_06
- OS : CentOS Linux release 7.5.1804 (Core) <== 최소설치 옵션을 사용하여 설치 하였음
- JDK :
  - openjdk version "1.8.0_181"
  - OpenJDK Runtime Environment (build 1.8.0_181-b13)
  - OpenJDK 64-Bit Server VM (build 25.181-b13, mixed mode)
- Elastic Stack : 6.4.0 ~ 6.5.4

# 전체 공통 설정
## SELinux 끄기
- root(suto 포함) 권한으로 실행되는 프로그램(Metricbeat, Nginx 등)이 있는 경우 끄고 사용하는 것이 좋음
- disabled 상태로 변경 한 후 다시 enforcing 상태로 되돌릴 경우 어떤 오류가 발생 할지 알 수 없음.
- permissive 상태는 모든 상태 로그는 그대로 남겨두기 때문에 다시 enforcing 상태로 변경 하여도 문제가 발생하지 않음.
### 명령어로 임시 조치
```shell
# 현재 상태 확인
~$ sudo sestatus

# permissive 상태로 임시 변경
~$ sudo setenforce 0
```

### 서버 설정 파일 편집
```shell
# selinux 파일 편집
~$ sudo vi /etc/sysconfig/selinux
-- 내용 수정 -----------------------------------------------------------------------------------
# This file controls the state of SELinux on the system.
# SELINUX= can take one of these three values:
#     enforcing - SELinux security policy is enforced.
#     permissive - SELinux prints warnings instead of enforcing.
#     disabled - No SELinux policy is loaded.
#SELINUX=enforcing <== 주석 처리
SELINUX=permissive <== 내용 추가
# SELINUXTYPE= can take one of three two values:
#     targeted - Targeted processes are protected,
#     minimum - Modification of targeted policy. Only selected processes are protected.
#     mls - Multi Level Security protection.
SELINUXTYPE=targeted
-- 내용 수정 -----------------------------------------------------------------------------------

# 서버 재기동
~$ sudo reboot 0
```

## Swap 끄기
- 꼭 필요한 경우가 아니면 가급적 끄지 말고 사용 할 것!
- 테스트 용도에서의 경우 굳이 Swap 설정을 끌 필요는 없음
- Elasticsearch 메뉴얼을 확인 해 보면 운영서버의 경우 끄라고 되어 있음
```shell
# swap 사용 중지
~$ sudo swapoff -a

# free-up 공간 제거
~$ sudo lvremove -Ay /dev/centos/swap

# /dev/centos/root 경로 재설정
~$ sudo lvextend -l +100%FREE /dev/centos/root

# grub2.cfg 파일 재생성 전 파일 수정
~$ sudo vi /etc/default/grub
-- 내용 수정 -----------------------------------------------------------------------------------
GRUB_TIMEOUT=5
GRUB_DEFAULT=saved
GRUB_DISABLE_SUBMENU=true
GRUB_TERMINAL_OUTPUT="console"
##GRUB_CMDLINE_LINUX="rd.lvm.lv=centos/root rd.lvm.lv=centos/swap crashkernel=auto rhgb quiet" <== 주석처리
GRUB_CMDLINE_LINUX="rd.lvm.lv=centos/root crashkernel=auto rhgb quiet" <== 내용추가
GRUB_DISABLE_RECOVERY="true"
-- 내용 수정 -----------------------------------------------------------------------------------

# 기존 파일 백업
~$ sudo cp /etc/grub2.cfg /etc/grub2.cfg.bak

# grub2.cfg 파일 재생성
~$ sudo grub2-mkconfig > /etc/grub2.cfg

# 서버 재기동
~$ sudo reboot 0
```

## JDK 설치
- JDK를 사용하는 모든 프로그램(Elasticsearch, Logstash 등)의 가상머신에 모두 설치 할 것
- java.policy 파일 수정은 Elasticsearch 실행 시 관련 에러가 발생하면 수정 해야 함
```shell
# JDK 확인
~$ sudo yum list java*jdk-devel
-- 출력 내용 -----------------------------------------------------------------------------------
Loaded plugins: fastestmirror
Determining fastest mirrors
 * base: mirror.kakao.com
 * epel: mirror.premi.st
 * extras: mirror.kakao.com
 * updates: mirror.kakao.com
Available Packages
java-1.6.0-openjdk-devel.x86_64 1:1.6.0.41-1.13.13.1.el7_3 base
java-1.7.0-openjdk-devel.x86_64 1:1.7.0.201-2.6.16.1.el7_6 updates
java-1.8.0-openjdk-devel.i686   1:1.8.0.191.b12-1.el7_6    updates
java-1.8.0-openjdk-devel.x86_64 1:1.8.0.191.b12-1.el7_6    updates <== 설치
java-11-openjdk-devel.i686      1:11.0.1.13-3.el7_6        updates
java-11-openjdk-devel.x86_64    1:11.0.1.13-3.el7_6        upd
-- 출력 내용 -----------------------------------------------------------------------------------

# 설치
~$ sudo yum install -y java-1.8.0-openjdk-devel.x86_64

# 설치 후 확인
~$ java -version
-- 출력 내용 -----------------------------------------------------------------------------------
openjdk version "1.8.0_181"
OpenJDK Runtime Environment (build 1.8.0_181-b13)
OpenJDK 64-Bit Server VM (build 25.181-b13, mixed mode)
-- 출력 내용 -----------------------------------------------------------------------------------

# java.policy 파일 찾기
~$ sudo find / -name 'java.policy'
-- 출력 내용 -----------------------------------------------------------------------------------
/usr/lib/jvm/java-1.8.0-openjdk-1.8.0.181-3.b13.el7_5.x86_64/jre/lib/security/java.policy
-- 출력 내용 -----------------------------------------------------------------------------------

# 파일 내용 확인
~$ cat /usr/lib/jvm/java-1.8.0-openjdk-1.8.0.181-3.b13.el7_5.x86_64/jre/lib/security/java.policy
-- 출력 내용 -----------------------------------------------------------------------------------
...... 내용 생략
permission java.io.FilePermission "<<ALL FILES>>", "read";
...... 내용 생략
-- 출력 내용 -----------------------------------------------------------------------------------
 
# 위 내용이 없다면 추가
~$ sudo vi /usr/lib/jvm/java-1.8.0-openjdk-1.8.0.181-3.b13.el7_5.x86_64/jre/lib/security/java.policy
-- 내용 수정 -----------------------------------------------------------------------------------
// Standard extensions get all permissions by default
 
grant codeBase "file:${{java.ext.dirs}}/*" {
        permission java.security.AllPermission;
};
 
// default permissions granted to all domains
 
grant {
...... 내용 생략
 
        permission java.io.FilePermission "<<ALL FILES>>", "read";
};
-- 내용 수정 -----------------------------------------------------------------------------------
```

## CentOS 7 최소설치 후 필요한 프로그램 설치
- 테스트에 사용할 6대의 가상머신에 모두 설치 할 것
```shell
# ifconfig 관련 설치
~$ sudo yum install -y net-tools

# jq 관련 설치 : Logstash 테스트 시 json 파일 포멧을 변경 할 필요 있는 경우에만 설치
~$ sudo yum install -y epel-release
~$ sudo yum install -y jq

# wget 설치
~$ sudo yum install -y wget

# shasum 설치 : tar.gz 파일을 내려받은 후 sha512 검증을 할 필요가 있는 경우에만 설치
~$ sudo yum install -y perl-Digest-SHA

# zip, unzip, bzip2 설치 : unzip의 경우는 인증서 파일 압축 해제 시 필요함, bzip2 는 Anaconda(Python)을 설치 하지 않으면 필요 없음
~$ sudo yum install -y zip
~$ sudo yum install -y unzip
~$ sudo yum install -y bzip2

# 개발툴 설치 : 간혹 특정 프로그램 설치 시 필요한 라이브러리들이 있어 미리 설치 해 두는 것이 좋음
~$ sudo yum update
~$ sudo yum groupinstall "Development Tools"

# firewall-cmd 설치 : 방화벽 포트 오픈 시 편하게 설정 할 수 있음
~$ sudo yum install firewalld
~$ sudo systemctl start firewalld
~$ sudo systemctl enable firewalld
```

## hosts 파일 수정
- Elasticsearch HOST 명의 경우 X-Pack 설정에서 인증서 생성 시 HOST 명에 “_” 를 사용하면 오류가 발생하므로 주의 할것!!
- 테스트에 사용할 6대의 가상머신에 모두 설정 할 것
```shell
# hosts 파일 확인
~$ cat /etc/hosts
-- 출력 내용 -----------------------------------------------------------------------------------
127.0.0.1   localhost localhost.localdomain localhost4 localhost4.localdomain4
::1         localhost localhost.localdomain localhost6 localhost6.localdomain6
-- 출력 내용 -----------------------------------------------------------------------------------

# hosts 파일에 IP / HOST 명 등록
~$ sudo vi /etc/hosts
-- 내용 수정 -----------------------------------------------------------------------------------
127.0.0.1   localhost localhost.localdomain localhost4 localhost4.localdomain4
::1         localhost localhost.localdomain localhost6 localhost6.localdomain6

192.168.56.101 elastic-01
192.168.56.102 elastic-02
192.168.56.103 elastic-03
192.168.56.104 kibana_04
192.168.56.105 logstash_05
192.168.56.106 beats_06
-- 내용 수정 -----------------------------------------------------------------------------------
```

# Elasticsearch 설치
- 3대의 가상머신(elastic-01, elastic-02, elastic-03)에 모두 설정 및 설치
## 시스템 설정 확인 및 수정
- yourid : Elasticsearch 를 실행하는 계정
```shell
# 기존 limits.conf 상태 확인
~$ cat /etc/security/limits.conf
-- 출력 내용 -----------------------------------------------------------------------------------
...... 내용 생략
yourid hard nofile 65536
yourid soft nofile 65536
yourid hard nproc 65536
yourid soft nproc 65536
yourid hard memlock unlimited
yourid soft memlock unlimited
-- 출력 내용 -----------------------------------------------------------------------------------
 
# 위 내용이 없다면 내용 추가
~$ sudo vi /etc/security/limits.conf
-- 내용 수정 -----------------------------------------------------------------------------------
#!/bin/bash
# THIS FILE IS ADDED FOR COMPATIBILITY PURPOSES
...... 내용 생략
#ftp             hard    nproc           0
#@student        -       maxlogins       4
 
yourid hard nofile 65536
yourid soft nofile 65536
yourid hard nproc 65536
yourid soft nproc 65536
yourid hard memlock unlimited
yourid soft memlock unlimited
 
# End of file
-- 내용 수정 -----------------------------------------------------------------------------------
 
# rc.local 내용 확인
~$ cat /etc/rc.local
-- 출력 내용 -----------------------------------------------------------------------------------
#!/bin/bash
# THIS FILE IS ADDED FOR COMPATIBILITY PURPOSES
#
# It is highly advisable to create own systemd services or udev rules
# to run scripts during boot instead of using this file.
#
# In contrast to previous versions due to parallel execution during boot
# this script will NOT be run after all other services.
#
# Please note that you must run 'chmod +x /etc/rc.d/rc.local' to ensure
# that this script will be executed during boot.

touch /var/lock/subsys/local
-- 출력 내용 -----------------------------------------------------------------------------------
 
# /proc/sys/vm/max_map_count 내용 확인
~$ cat /proc/sys/vm/max_map_count

# /proc/sys/fs/file-max 내용 확인
~$ cat /proc/sys/fs/file-max

# 값이 262144 미만인 경우 rc.local 파일에 내용 추가
~$ sudo vi /etc/rc.local
-- 내용 수정 -----------------------------------------------------------------------------------
#!/bin/bash
# THIS FILE IS ADDED FOR COMPATIBILITY PURPOSES
...... 내용 생략
# that this script will be executed during boot.
 
touch /var/lock/subsys/local
 
echo 1048575 > /proc/sys/vm/max_map_count <== 추가
echo 262144 > /proc/sys/fs/file-max <== 추가
-- 내용 수정 -----------------------------------------------------------------------------------
 
# rc.local 파일이 수정된 경우 /etc/rc.d/rc.local 파일에 실행 권한 추가
~$ ls -al /etc/rc.d/rc.local
-- 출력 내용 -----------------------------------------------------------------------------------
-rwxr-xr-x. 1 root root 565  8월 20 16:15 /etc/rc.d/rc.local
-- 출력 내용 -----------------------------------------------------------------------------------

# 실행 권한이 없는 경우 권한 추가
~$ sudo chmod +x /etc/rc.d/rc.local
 
# 지금까지의 변경 내용이 있는 경우 서버 재기동 필요
~$ sudo reboot 0
```

## Elasticsearch 내려받기
- elastic-01, elastic-02, elastic-03 에서 모두 동일하게 내려받기
```shell
# ElasticSearch 6.4 다운로드 및 파일 확인
~$ wget https://artifacts.elastic.co/downloads/elasticsearch/elasticsearch-6.4.0.tar.gz
~$ wget https://artifacts.elastic.co/downloads/elasticsearch/elasticsearch-6.4.0.tar.gz.sha512
~$ shasum -a 512 -c elasticsearch-6.4.0.tar.gz.sha512
 
# 압축 해제
~$ tar -xzf elasticsearch-6.4.0.tar.gz

# 심볼릭 링크 생성
~$ ln -s elasticsearch-6.4.0/ elasticsearch
```

## Elasticsearch 설정 파일 수정
- cluster.name 은 3대의 서버에 모두 동일하게 설정
  - 하나의 Cluster로 묶기 위함
- node.name 은 3대의 서버에 각각 다른 이름으로 설정
  - elastic-01 : node-1
  - elastic-02 : node-2
  - elastic-03 : node-3
- network.host 은 3대의 서버에 각각의 서버 IP로 설정
  - elastic-01 : 192.168.56.101
  - elastic-02 : 192.168.56.102
  - elastic-03 : 192.168.56.103
- network.host 설정 시 “\_site\_” 로 설정 해도 됨
  - 추후 인증서 생성 시에는 반드시 /etc/hosts 파일에 설정된 IP, HOST 명을 사용해야함
```shell
# elasticsearch.yml 파일 수정
~$ vi elasticsearch/config/elasticsearch.yml
-- 내용 수정 -----------------------------------------------------------------------------------
# ======================== Elasticsearch Configuration =========================
#
...... 내용 생략
#
# ---------------------------------- Cluster -----------------------------------
#
# Use a descriptive name for your cluster:
#
cluster.name: local-cluster <== 주석 해제 및 수정
#
# ------------------------------------ Node ------------------------------------
#
# Use a descriptive name for the node:
#
node.name: node-1 <== 주석 해제 및 수정
 
node.master: true <== 내용 추가
node.data: true <== 내용 추가
node.ingest: true <== 내용 추가
#
# Add custom attributes to the node:
#
#node.attr.rack: r1
#
# ----------------------------------- Paths ------------------------------------
#
# Path to directory where to store the data (separate multiple locations by comma):
#
#path.data: /path/to/data
#
# Path to log files:
#
#path.logs: /path/to/data
#
# ----------------------------------- Memory -----------------------------------
#
# Lock the memory on startup:
#
bootstrap.memory_lock: true <== 주석 해제 및 수정
#
...... 내용 생략
#
# ---------------------------------- Network -----------------------------------
#
# Set the bind address to a specific IP (IPv4 or IPv6):
#
network.host: 192.168.56.101 <== 주석 해제 및 수정
#
# Set a custom port for HTTP:
#
http.port: 9200 <== 주석 해제 및 수정
#
# For more information, consult the network module documentation.
#
# --------------------------------- Discovery ----------------------------------
#
# Pass an initial list of hosts to perform discovery when new node is started:
# The default list of hosts is ["127.0.0.1", "[::1]"]
#
discovery.zen.ping.unicast.hosts: ["elastic-01", "elastic-02", "elastic-03"] <== 주석 해제 및 수정
#
# Prevent the "split brain" by configuring the majority of nodes (total number of master-eligible nodes / 2 + 1):
...... 내용 생략
#
#action.destructive_requires_name: true
-- 내용 수정 -----------------------------------------------------------------------------------
 
# jvm.options 파일 수정
~$ vi elasticsearch/config/jvm.options
-- 내용 수정 -----------------------------------------------------------------------------------
## JVM configuration
 
################################################################
## IMPORTANT: JVM heap size
...... 내용 생략
################################################################
 
# Xms represents the initial size of total heap space
# Xmx represents the maximum size of total heap space
 
-Xms2g <== 내용 수정
-Xmx2g <== 내용 수정
 
################################################################
## Expert settings
...... 내용 생략
 
# temporary workaround for C2 bug with JDK 10 on hardware with AVX-512
10-:-XX:UseAVX=2
-- 내용 수정 -----------------------------------------------------------------------------------
```

## 한글형태소 분석기(Nori) 설치
- elastic-01, elastic-02, elastic-03 에서 모두 동일하게 설치
```shell
# 작업 디렉토리 이동
~$ cd elasticsearch
 
# Nori 형태소 분석기 plugin 설치
elasticsearch$ bin/elasticsearch-plugin install analysis-nori
```

## start.sh, stop.sh 생성
- elastic-01, elastic-02, elastic-03 에서 모두 동일하게 생성
- 실행 / 중지를 편하게 하기 위한 목적으로 필요 없을 경우 생략해도 무방함
```shell
# start.sh 스크립트 작성
elasticsearch$ cat > start.sh
bin/elasticsearch -d -p es.pid
# Ctrl + C 를 눌러서 빠져나옴
 
# stop.sh 스크립트 작성
elasticsearch$ cat > stop.sh
kill `cat es.pid`
# Ctrl + C 를 눌러서 빠져나옴
 
# 작성한 스크립트 실행 권한 추가
elasticsearch$ chmod +x *.sh
```

## 방화벽 설정
- 방화벽 데몬을 재실행 해야 적용이 완료됨
```shell
# 9200 포트 열기
elasticsearch$ sudo firewall-cmd --permanent --add-port=9200/tcp

# 9300 포트 열기
elasticsearch$ sudo firewall-cmd --permanent --add-port=9300/tcp

# 방화벽 설정 적용
elasticsearch$ sudo firewall-cmd --reload

# 방화벽 설정 확인
elasticsearch$ sudo firewall-cmd --list-all

# 방화벽 데몬 재실행
elasticsearch$ sudo systemctl restart firewalld
```

## Elasticsearch 실행 / 중지
```shell
# 실행
elasticsearch$ ./start.sh

# 중지
elasticsearch$ ./stop.sh
```

## Elasticsearch 노드간 통신 TLS 적용을 위한 인증서 생성
- elastic-01 에서 생성 한 후 elastic-02, elastic-03 에 복사하여 동일한 파일로 설정 해야함
- 6.3.x 버전 이후부터는 TLS 적용이 가능하며, X-Pack을 적용하지 않아도 사용 가능함
  - TLS 를 적용해야 라이센스 적용 및 추후 X-Pack 적용이 쉬움
  - TLS 설정이 되어 있지 않은 경우 라이센스 적용이 되지 않음
- instance name 은 elasticsearch.yml 파일의 cluster.name 을 입력
- IP, DNS(HOST 명) 입력 시 잘못 작성하지 않도록 주의 해야함
  - /etc/hosts 파일에 명시된 이름 혹은 DNS 서버에서 확인 할 수 있는 이름으로 사용 해야함
```shell
# 인증서 생성
elasticsearch$ bin/elasticsearch-certgen
-- 출력 내용 -----------------------------------------------------------------------------------
...
Let's get started...
 
Please enter the desired output file [certificate-bundle.zip]: <== 엔터
Enter instance name: local-cluster <== 입력
Enter name for directories and files [local-cluster]: <== 엔터
Enter IP Addresses for instance (comma-separated if more than one) []: 192.168.56.101,192.168.56.102,192.168.56.103 <== 입력
Enter DNS names for instance (comma-separated if more than one) []: elastic-01,elastic-02,elastic-03 <== 입력
Would you like to specify another instance? Press 'y' to continue entering instance information: n <== 입력
Certificates written to /home/yourid/elasticsearch/certificate-bundle.zip
-- 출력 내용 -----------------------------------------------------------------------------------
 
# 생성된 인증서 압축 파일을 모든 노드 서버의 ~/elasticsearch 위치에 복사
elasticsearch$ scp certificate-bundle.zip yourid@elastic-02:~/elasticsearch
elasticsearch$ scp certificate-bundle.zip yourid@elastic-03:~/elasticsearch
```

## 인증서 적용
- elastic-01, elastic-02, elastic-03 에서 모두 동일하게 적용 후 재실행
```shell
# 인증서 보관 디렉토리 생성
elasticsearch$ mkdir cert
 
# 인증서 파일 이동
elasticsearch$ mv certificate-bundle.zip ./cert
 
# 작업 디렉토리 이동
elasticsearch$ cd cert

# 압축 해제
cert$ unzip certificate-bundle.zip

# 작업 디렉토리 이동
cert$ ~/elasticsearch/config
 
# elasticsearch.yml 파일 수정
config$ vi elasticsearch.yml
-- 내용 수정 -----------------------------------------------------------------------------------
# ======================== Elasticsearch Configuration =========================
#
...... 내용 생략
#action.destructive_requires_name: true
 
#Add options
xpack.security.enabled: false <== X-Pack 은 적용하지 않기 위해 false 로 설정
xpack.ssl.key: /home/yourid/elasticsearch/cert/local-cluster/local-cluster.key <== 디렉토리 위치 확인
xpack.ssl.certificate: /home/yourid/elasticsearch/cert/local-cluster/local-cluster.crt <== 디렉토리 위치 확인
xpack.ssl.certificate_authorities: [ "/home/yourid/elasticsearch/cert/ca/ca.crt" ] <== 디렉토리 위치 확인
xpack.security.transport.ssl.enabled: true
-- 내용 수정 -----------------------------------------------------------------------------------

# 디렉토리 이동
config$ cd ~/elasticsearch
 
# 인증서 적용 후 모든 노드 서버 재기동
elasticsearch$ stop.sh
elasticsearch$ start.sh
```

# Kibana 설치
- kibana_04 서버에 설치
## Kibana 내려받기
```shell
# Kibana 6.4 다운로드 및 확인
~$ wget https://artifacts.elastic.co/downloads/kibana/kibana-6.4.0-linux-x86_64.tar.gz
~$ shasum -a 512 kibana-6.4.0-linux-x86_64.tar.gz
 
# 압축 해제
~$ tar -xzf kibana-6.4.0-linux-x86_64.tar.gz

# 심볼릭 링크 생성
~$ ln -s kibana-6.4.0-linux-x86_64/ kibana
```

## Kibana 설정 파일 수정
```shell
# kibana.yml 파일 수정
~$ vi kibana/config/kibana.yml
-- 내용 수정 -----------------------------------------------------------------------------------
# Kibana is served by a back end server. This setting specifies the port to use.
#server.port: 5601
 
...... 내용 생략
# To allow connections from remote users, set this parameter to a non-loopback address.
server.host: "0.0.0.0" <== 주석 해제 및 수정
 
# Enables you to specify a path to mount Kibana at if you are running behind a proxy.
...... 내용 생략
 
# The URL of the Elasticsearch instance to use for all your queries.
elasticsearch.url: "http://elastic-01:9200" <== 주석 해제 및 수정
 
# When this setting's value is true Kibana uses the hostname specified in the server.host
...... 내용 생략
# Specifies the path where Kibana creates the process ID file.
pid.file: kibana.pid <== 주석 해제 및 수정
 
# Enables you specify a file where Kibana stores log output.
...... 내용 생략
#i18n.defaultLocale: "en"
-- 내용 수정 -----------------------------------------------------------------------------------
```

## start.sh, stop.sh 생성
- 실행 / 중지를 편하게 하기 위한 목적으로 필요 없을 경우 생략해도 무방함
```shell
# 작업 디렉토리 이동
~$ cd kibana
 
# start.sh 스크립트 작성
kibana$ cat > start.sh
sudo logrotate -f /etc/logrotate.d/kibana <== logrotate 설정은 아래의 내용 참고
nohup bin/kibana > /var/log/kibana/kibana.log 2>&1 & <== Console 에 나오는 Log를 파일로 남김
# Ctrl + C 를 눌러서 빠져나옴
 
# stop.sh 스크립트 작성
kibana$ cat > stop.sh
#!/bin/bash
kill `cat kibana.pid`
  
if [ -f "kibana.pid" ] ; then
    rm kibana.pid
fi
# Ctrl + C 를 눌러서 빠져나옴
 
# 실행권한 추가
kibana$ chmod +x *.sh
```

## Kibana Log 관리를 위한 logrotate 설정
- Kibana의 경우 Node.js 로 만들어져서 Java로 만들어진 Elasticsearch, Logstash에 비해 자체 Log 처리가 조금 미흡함
- 서버 Console 에 나오는 Log를 파일로 관리 하고 싶은 경우 참고 할 것
```shell
# 작업 디렉토리 이동
kibana$ cd /var/log
 
# Kibana log 디렉토리 생성 및 소유권 변경
log$ sudo mkdir kibana
log$ sudo chown yourid:yourid kibana
 
# 작업 디렉토리 이동
log$ cd /etc/logrotate.d
 
# logrotate 설정 추가
logrotate.d$ sudo vi kibana
-- 내용 작성 -----------------------------------------------------------------------------------
/var/log/kibana/kibana.log {
    copytruncate
    daily
    rotate 15
    missingok
    notifempty
    compress
    dateext
    dateformat -%Y%m%d_%s
    postrotate
        /bin/chown yourid:yourid /var/log/kibana/kibana.log*
    endscript
    su root yourid
}
-- 내용 작성 -----------------------------------------------------------------------------------
```

## 방화벽 설정
- 방화벽 데몬을 재실행 해야 적용이 완료됨
```shell
# 5601 포트 열기
logrotate.d$ sudo firewall-cmd --permanent --add-port=5601/tcp

# 방화벽 설정 적용
logrotate.d$ sudo firewall-cmd --reload

# 방화벽 설정 확인
logrotate.d$ sudo firewall-cmd --list-all

# 방화벽 데몬 재실행
logrotate.d$ sudo systemctl restart firewalld
```

## Kibana 실행 / 중지
```shell
# 작업 디렉토리 이동
logrotate.d$ cd ~/kibana

# 실행
kibana$ ./start.sh

# 중지
kibana$ ./stop.sh
```

# Logstash 설치
- logstash_05 서버에 설치
## Logstash 내려받기
```shell
# Logstash 6.4 다운로드 및 확인
~$ wget https://artifacts.elastic.co/downloads/logstash/logstash-6.4.0.tar.gz
~$ wget https://artifacts.elastic.co/downloads/logstash/logstash-6.4.0.tar.gz.sha512
~$ shasum -a 512 -c logstash-6.4.0.tar.gz.sha512
 
# 압축 해제
~$ tar -xzf logstash-6.4.0.tar.gz
 
# 심볼릭 링크 변경
~$ ln -s logstash-6.4.0/ logstash
```

## Sample conf 파일 생성
- 실제 동작하는 예제는 아니며, .conf 파일을 여러개 사용하여 하나의 서버에서 동시에 관리 하는 방법을 위한 설정 예제임
- 명령어가 매우 길어지므로 start.sh, stop.sh 파일을 생성하여 관리하는 것이 편함
  - 프로세스 종료 시 “pkill -ef” 명령어를 사용하므로 conf 파일 생성 시 파일명을 구분 가능하도록 생성 해야함
- Sample 작업을 위한 디렉토리 구조
```
-- logstash
 |-- conf-file
   |-- sample1
     |-- sample1.conf
     |-- start.sh
     |-- stop.sh
   |-- sample2
     |-- sample2.conf
     |-- start.sh
     |-- stop.sh
```
```shell
# 작업 디렉토리 이동
~$ cd logstash

# .conf 파일들을 보관할 디렉토리 생성
logstash$ mkdir conf-file

# 작업 디렉토리 이동
logstash$ cd conf-file

# sample 디렉토리 생성
conf-file$ mkdir sample1
conf-file$ mkdir sample2

# 작업 디렉토리 이동
conf-file$ cd sample1

# sample1.conf 파일 생성
sample1$ vi sample1.conf
-- 내용 작성 -----------------------------------------------------------------------------------
input {
  file {
    path => "/home/yourid/logstash/conf-file/sample1/network_traffic_data1.json"
    start_position => "beginning"
    sincedb_path => "/dev/null"
    codec => json {
      charset => "ISO-8859-1"
    }
  }
}
output {
  elasticsearch {
    hosts => ["elastic-01:9200", "elastic-02:9200", "elastic-03:9200"]
    index => "bigginsight"
    document_type => "usageReport"
  }
  stdout {}
}
-- 내용 작성 -----------------------------------------------------------------------------------

# 작업 디렉토리 이동
sample1$ cd ../sample2

# sample2.conf 파일 생성
sample2$ vi sample2.conf
-- 내용 작성 -----------------------------------------------------------------------------------
input {
  file {
    path => "/home/yourid/logstash/conf-file/sample2/network_traffic_data2.json"
    start_position => "beginning"
    sincedb_path => "/dev/null"
    codec => json {
      charset => "ISO-8859-1"
    }
  }
}
output {
  elasticsearch {
    hosts => ["elastic-01:9200", "elastic-02:9200", "elastic-03:9200"]
    index => "bigginsight"
    document_type => "usageReport"
  }
  stdout {}
}
-- 내용 작성 -----------------------------------------------------------------------------------
```

### Sample1 start.sh, stop.sh 파일 생성
```shell
# 작업 디렉토리 이동
sample2$ cd ../sample1

# start.sh 파일 생성
sample1$ vi start.sh
-- 내용 작성 -----------------------------------------------------------------------------------
nohup ~/logstash/bin/logstash -n logstash_06-sample1 -f sample1.conf -l /home/yourid/logstash/logs/sample1 --path.data /home/yourid/logstash/data/sample1 --log.level=debug > /dev/null &
-- 내용 작성 -----------------------------------------------------------------------------------

# stop.sh 파일 생성
sample1$ vi stop.sh
-- 내용 작성 -----------------------------------------------------------------------------------
pkill -ef sample1.conf
-- 내용 작성 -----------------------------------------------------------------------------------

# 실행 권한 추가
sample1$ chmod +x *sh
```

### Sample2 start.sh, stop.sh 파일 생성
```shell
# 작업 디렉토리 이동
sample1$ cd ../sample2

# start.sh 파일 생성
sample2$ vi start.sh
-- 내용 작성 -----------------------------------------------------------------------------------
nohup ~/logstash/bin/logstash -n logstash_06-sample2 -f sample2.conf -l /home/yourid/logstash/logs/sample2 --path.data /home/yourid/logstash/data/sample2 --log.level=debug > /dev/null &
-- 내용 작성 -----------------------------------------------------------------------------------

# stop.sh 파일 생성
sample2$ vi stop.sh
-- 내용 작성 -----------------------------------------------------------------------------------
pkill -ef sample2.conf
-- 내용 작성 -----------------------------------------------------------------------------------

# 실행 권한 추가
sample2$ chmod +x *sh
```

## start.sh 파일 내용 설명
- nohup : 실행 시 Console에 로그를 출력하지 않도록 하는 명령어
- -n [NAME] : 개별로 실행하는 경우 각 Logstash instance를 구분하기 위한 이름(--node.name NAME)
- -f [FILE_PATH] : .conf 파일 위치
- -l [DIR_PATH] : Log 디렉토리 위치
- --path.data [DIR_PATH] : Data 디렉토리 위치
- --log.level=[LEVEL] : 기본값은 “info”
- \> /dev/null : stdout 을 null 로 보내도록 하는 유닉스 명령어
- & : 백그라운드로 실행 시키는 유닉스 명령어
- 여기서 중요한 부분은 “-n”, “-l”, “--path.data” 임
  - Logstash instance 이름을 넣지 않으면 모두 동일한 이름으로 설정되어 추후 X-Pack 모니터링 시 구분이 안됨
  - Log 디렉토리 위치가 동일한 경우 모두 하나의 Log 파일(logstash-plain.log)에 Log 가 생성되어 Log 구분이 어려움
  - **Data 디렉토리 위치가 동일한 경우 Multi Logstash instance 를 실행 하는 경우 Data 디렉토리가 겹치면 안되다는 오류가 발생하면서 중지됨**
- 보다 자세한 내용은 Logstash 영문 메뉴얼의 “Running Logstash from the Command Line” 부분을 참고 할 것

# 참고
- https://www.lesstif.com/pages/viewpage.action?pageId=6979732
- https://www.refmanual.com/2016/01/08/completely-remove-swap-on-ce7/
- https://www.elastic.co/guide/en/elasticsearch/reference/current/system-config.html
- https://www.elastic.co/guide/en/elasticsearch/reference/current/zip-targz.html
- https://www.elastic.co/guide/en/elasticsearch/reference/current/bootstrap-checks.html
- http://kimjmin.net/2018/08/2018-08-install-security-over-es63/
- http://blueskai.tistory.com/101
- https://www.elastic.co/guide/en/logstash/current/installing-logstash.html
- https://www.elastic.co/guide/en/logstash/current/running-logstash-command-line.html
