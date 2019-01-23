# Elastic Stack X-Pack 적용 가이드
- 이 문서에서는 Elasticsearch 6.4.0 기준으로 적용이 진행 되지만, 6.5.4 버전도 동일하게 적용 할 수 있음

# 테스트 환경
[Elastic Stack 설치 가이드](Elasticsearch설치.md) 문서 참조

# 적용전 확인 사항
- 수집되고 있는 데이터 파이프라인이 있으면, 모두 중지한다.
  - X-Pack 적용 도중 Elasticsearch 를 순차 적용 할 수는 있으나, 계정설정, SSL 적용 시 기존에 수집되고 있던 Logstash, Metricbeat 등에서 접속 오류가 발생한다.
  - X-Pack 을 적용함과 동시에 모든 접근 Client에 계정정보 및 SSL 인증을 요구하게 되므로 반드시 데이터 수집 중지 후 작업 하는 것이 정신 건강에 이로움
  - 특히 Kibana 역시 계정설정 및 SSL 적용을 하지 않으면 Elasticsearch 에 접속이 불가능함
- Elasticsearch에 직접 접속하는 별도의 Client가 있다면, 작업중 혹은 작업 이후에 접속이 불가능 함을 공지한다.
  - X-Pack 적용 완료 후 SSL 접속 설정 및 접속 시 계정정보 설정을 각 Client 상황에 맞게 별도 설정 해야한다.
  - 여기서는 기존에 설정 했던 Kibana, Logstash, Metricbeat 에 대해서는 계정 설정 및 SSL 설정을 진행 한다.
  - Kibana, Logstash, Metricbeat 설정을 참고하여 각 Client 의 상황에 맞게 설정을 해야한다.
  - 주요 설정 사항은 아래와 같음
    - Elasticsearch 접속 시 http 에서 https 로 변경
    - 인증서 경고를 무시하는 설정 필요
    - Elasticsearch 접속 시 반드시 접속 계정정보(ID / PW)를 설정 해야함
- Kibana 에서 30일 Trial license 를 적용한다.
  - 유료 라이센스가 있다면 해당 라이센스를 적용 해도 된다.
  - 미리 적용하는 이유는 Elasticsearch 에 X-Pack 설정을 적용하면, Kibana 에도 X-Pack 관련 설정을 하지 않으면 접속이 불가능 하기 때문임

# Elasticsearch, Kibana X-Pack 적용
- 작업 진행 전 가급적 모든 접속 Client 들을 중지 시킨다.
- Elasticsearch 와 Kibana는 따로 적용하기 보단 서로간의 연관성으로 같이 진행됨
## Elasticsearch 설정 파일 수정 - X-Pack 적용
- 3대의 가상머신(elastic-01, elastic-02, elastic-03)에 모두 설정
- X-Pack 설정을 true 로 TLS 설정을 false 로 변경
  - X-Pack 설정 시 TLS 설정이 있는 경우 설정된 계정정보 전파가 안되므로 TLS 설정을 끄고 계정설정을 해야함
```shell
# elasticsearch.yml 파일 수정
~$ vi elasticsearch/config/elasticsearch.yml
-- 내용 수정 -----------------------------------------------------------------------------------
내용생략....

#Add options
xpack.security.enabled: true <== false 에서 true 로 변경
xpack.ssl.key: /home/yourid/elasticsearch/cert/local-cluster-1/local-cluster-1.key
xpack.ssl.certificate: /home/yourid/elasticsearch/cert/local-cluster-1/local-cluster-1.crt
xpack.ssl.certificate_authorities: [ "/home/yourid/elasticsearch/cert/ca/ca.crt" ]
xpack.security.transport.ssl.enabled: false <== true 에서 false 로 변경
-- 내용 수정 -----------------------------------------------------------------------------------
```

## Elasticsearch 순차 재실행
- 이 부분은 Rolling upgrade 방식과 거의 동일함
### Shard Allocation 중지
- Kibana -> Dev Tools 에서 실행
```shell
PUT _cluster/settings
{
  "persistent": {
    "cluster.routing.allocation.enable": "none"
  }
}
```

### Sync Flus 실행
- Kibana -> Dev Tools 에서 실행
- 실행 결과에서 fail 수가 0 일 때까지 적당히 진행 후 이후 작업을 진행
```shell
POST _flush/synced
```

### Elasticsearch 재실행
- elastic-01 서버만 재실행
```shell
# 작업 디렉토리 이동
~$ cd elasticsearch

# 중지
elasticsearch$ ./stop.sh

# 실행
elasticsearch$ ./start.sh
```

### Shard Allocation 재가동
- Kibana -> Dev Tools 에서 실행
- Kibana 에서 상태를 모니터링 하면서 GREEN 으로 될때까지 기다림
```shell
PUT _cluster/settings
{
  "persistent": {
    "cluster.routing.allocation.enable": null
  }
}
```

### 나머지 서버 순차 재실행
- elastic-02, elastic-03 서버도 elastic-01 서버에서 실행한 단계를 반복
  - Shard Allocation 중지
  - Sync Flus 실행
  - Elasticsearch 재실행
  - Shard Allocation 재가동

### Elasticsearch X-Pack 계정설정
- elastic-01 서버에서만 진행함
  - 계정설정은 모든 Node 서버가 아닌 한대의 서버에서 설정하여 전파되는 구조임
  - 여기서는 편의상 모든 비밀번호는 changeme 로 설정
#### 6.4.x 의 경우
```shell
# X-Pack 계정 생성
elasticsearch$ bin/elasticsearch-setup-passwords interactive
-- 출력 내용 -----------------------------------------------------------------------------------
Initiating the setup of passwords for reserved users elastic,kibana,logstash_system,beats_system.
You will be prompted to enter passwords as the process progresses.
Please confirm that you would like to continue [y/N]y

Enter password for [elastic]: <== changeme 입력
Reenter password for [elastic]: <== changeme 입력
Enter password for [kibana]: <== changeme 입력
Reenter password for [kibana]: <== changeme 입력
Enter password for [logstash_system]: <== changeme 입력
Reenter password for [logstash_system]: <== changeme 입력
Enter password for [beats_system]: <== changeme 입력
Reenter password for [beats_system]: <== changeme 입력
Changed password for user [kibana]
Changed password for user [logstash_system]
Changed password for user [beats_system]
Changed password for user [elastic]
-- 출력 내용 -----------------------------------------------------------------------------------
```

#### 6.5.x 의 경우
```shell
# X-Pack 계정 생성
elasticsearch$ bin/elasticsearch-setup-passwords interactive
-- 출력 내용 -----------------------------------------------------------------------------------
Initiating the setup of passwords for reserved users elastic,apm_system,kibana,logstash_system,beats_system,remote_monitoring_user.
You will be prompted to enter passwords as the process progresses.
Please confirm that you would like to continue [y/N]y

Enter password for [elastic]: <== changeme 입력
Reenter password for [elastic]: <== changeme 입력
Enter password for [apm_system]: <== changeme 입력
Reenter password for [apm_system]: <== changeme 입력
Enter password for [kibana]: <== changeme 입력
Reenter password for [kibana]: <== changeme 입력
Enter password for [logstash_system]: <== changeme 입력
Reenter password for [logstash_system]: <== changeme 입력
Enter password for [beats_system]: <== changeme 입력
Reenter password for [beats_system]: <== changeme 입력
Enter password for [remote_monitoring_user]: <== changeme 입력
Reenter password for [remote_monitoring_user]: <== changeme 입력
Changed password for user [apm_system]
Changed password for user [kibana]
Changed password for user [logstash_system]
Changed password for user [beats_system]
Changed password for user [remote_monitoring_user]
Changed password for user [elastic]
-- 출력 내용 -----------------------------------------------------------------------------------
```

## Elasticsearch 설정 파일 수정 - TLS, SSL 적용
- 3대의 가상머신(elastic-01, elastic-02, elastic-03)에 모두 설정
```shell
# elasticsearch.yml 파일 수정
elasticsearch$ vi config/elasticsearch.yml
-- 내용 수정 -----------------------------------------------------------------------------------
내용생략....

#Add options
xpack.security.enabled: true
xpack.ssl.key: /home/yourid/elasticsearch/cert/local-cluster-1/local-cluster-1.key
xpack.ssl.certificate: /home/yourid/elasticsearch/cert/local-cluster-1/local-cluster-1.crt
xpack.ssl.certificate_authorities: [ "/home/yourid/elasticsearch/cert/ca/ca.crt" ]
xpack.security.transport.ssl.enabled: true <== false 에서 true 로 변경
xpack.security.http.ssl.enabled: true <== 추가
-- 내용 수정 -----------------------------------------------------------------------------------
```

## Elasticsearch 순차 재실행
- Shard Allocation 중지 및 Sync Flus 진행 이후 전체 Elasticsearch 재실행이 필요하므로 Rolling upgrade 방식과 유사하지만 다르다는 점에 유의 해야함
- 중지 시켰던 Shard Allocation 은 마지막 단계에서 다시 실행
- TLS 와 SSL 이 적용되면 각 Node 서버와 보안 통신을 하려고 하므로 서버 재실행 시 Log 에 많은 오류가 나타남
  - TLS, SSL 이 적용된 서버와 적용이 되지 않은 서버간의 통신은 당연히 되지 않음
  - 이 단계에서 외부와 통신하던 부분도 통신이 되지 않으므로 데이터을 주고 받는 모든 서버에서 오류가 발생함
  - 당연히 수집 데이터가 수집되지 않음
### Shard Allocation 중지
- Kibana -> Dev Tools 에서 실행
```shell
PUT _cluster/settings
{
  "persistent": {
    "cluster.routing.allocation.enable": "none"
  }
}
```

### Sync Flus 실행
- Kibana -> Dev Tools 에서 실행
- 실행 결과에서 fail 수가 0 일 때까지 적당히 진행 후 이후 작업을 진행
```shell
POST _flush/synced
```

### Elasticsearch 재실행
- 3대의 가상머신(elastic-01, elastic-02, elastic-03)을 모두 재실행
- TLS 와 SSL 이 적용되어 기존의 Kibana 에서는 이제 접속이 불가능함
```shell
# 중지
elasticsearch$ ./stop.sh

# 실행
elasticsearch$ ./start.sh
```

### Kibana 설정 수정
- requestTimeout 부분은 기본값이 부족하다고 느낄때만 수정 할 것
- verificationMode 부분은 인증서 경고를 무시 하도록함
  - Elasticsearch 에서 제공하는 인증서의 경우는 사설 인증서이므로 인증서 경고가 발생함
```shell
# kibana.yml 파일 수정
~$ vi kibana/config/kibana.yml
-- 내용 수정 -----------------------------------------------------------------------------------
내용생략....
# The URL of the Elasticsearch instance to use for all your queries.
elasticsearch.url: "https://elastic-01:9200" <== http 에서 https 로 변경
내용생략....
# index at startup. Your Kibana users still need to authenticate with Elasticsearch, which
# is proxied through the Kibana server.
elasticsearch.username: "kibana" <== 주석해제 및 수정
elasticsearch.password: "changeme" <== 주석해제 및 수정
내용생략....
# To disregard the validity of SSL certificates, change this setting's value to 'none'.
#elasticsearch.ssl.verificationMode: full
elasticsearch.ssl.verificationMode: none <== 추가
내용생략....
# Time in milliseconds to wait for responses from the back end or Elasticsearch. This value
# must be a positive integer.
#elasticsearch.requestTimeout: 30000
elasticsearch.requestTimeout: 90000 <== 추가
내용생략....
-- 내용 수정 -----------------------------------------------------------------------------------
```

### Kibana 재실행
- 재실행 후 Kibana 접속 시 로그인 화면이 나오면 성공
- Kibana 접속 계정은 elastic / changeme 로 접속
```shell
# 작업 디렉토리 이동
~$ cd kibana

# 중지
kibana$ ./stop.sh

# 실행
kibana$ ./start.sh
```

### Shard Allocation 재가동
- Kibana -> Dev Tools 에서 실행
- Kibana 에서 상태를 모니터링 하면서 GREEN 으로 될때까지 기다림
```shell
PUT _cluster/settings
{
  "persistent": {
    "cluster.routing.allocation.enable": null
  }
}
```

# Logstash X-Pack 적용
- Elasticsearch X-Pack 적용을 위해 미리 중지되었다고 가정함
- 모든 설정 부분이 아닌 X-Pack 관련 설정만 어떻게 수정하는지 작성함
## Elasticsearch 인증서 복사
```shell
# 작업 디렉토리 이동
elasticsearch$ cd cert

# 생성된 인증서 압축 파일을 모든 노드 서버의 ~/logstash 위치에 복사
cert$ scp certificate-bundle.zip yourid@logstash_05:~/logstash
```

## Logstash 인증서 압축 해제
```shell
# 작업 디렉토리 이동
~$ cd logstash

# 인증서 디렉토리 생성
logstash$ mkdir cert

# 인증서 압축파일 이동
logstash$ mv certificate-bundle.zip ./cert

# 작업 디렉토리 이동
logstash$ cd cert

# 압축 해제
cert$ unzip certificate-bundle.zip
```

## Logstash 설정 파일 수정
- X-Pack 모니터링을 위한 설정으로 설정 파일은 6.5.4 기준임
- 6.5.4 이전 버전의 경우 아래의 내용을 참고하여 최대한 비슷하게 수정 하면 될 듯 함
```shell
# 작업 디렉토리 이동
cert$ cd ..

# x-pack 모니터링 설정을 위한 logstash.yml 파일 수정
logstash$ vi config/logstash.yml
-- 내용 수정 -----------------------------------------------------------------------------------
내용생략....
# ------------ X-Pack Settings (not applicable for OSS build)--------------
#
# X-Pack Monitoring
# https://www.elastic.co/guide/en/logstash/current/monitoring-logstash.html
#xpack.monitoring.enabled: false
xpack.monitoring.enabled: true <== 추가 : X-Pack 모니터링 enabled
#xpack.monitoring.elasticsearch.username: logstash_system
#xpack.monitoring.elasticsearch.password: password
xpack.monitoring.elasticsearch.username: logstash_system <== 추가
xpack.monitoring.elasticsearch.password: changeme <== 추가
#xpack.monitoring.elasticsearch.url: ["https://es1:9200", "https://es2:9200"]
xpack.monitoring.elasticsearch.url: ["https://elastic-01:9200", "https://elastic-02:9200", "https://elastic-03:9200"] <== 추가 : Elasticsearch Url 정보
#xpack.monitoring.elasticsearch.ssl.ca: [ "/path/to/ca.crt" ]
xpack.monitoring.elasticsearch.ssl.ca: "/home/yourid/logstash/cert/ca/ca.crt" <== 추가 : SSL CA 인증서 설정, 없는 경우 Elasticsearch 와 모니터링 통신시 인증서 오류 발생
#xpack.monitoring.elasticsearch.ssl.truststore.path: path/to/file
#xpack.monitoring.elasticsearch.ssl.truststore.password: password
#xpack.monitoring.elasticsearch.ssl.keystore.path: /path/to/file
#xpack.monitoring.elasticsearch.ssl.keystore.password: password
#xpack.monitoring.elasticsearch.ssl.verification_mode: certificate
xpack.monitoring.elasticsearch.ssl.verification_mode: none <== 추가 : SSL 인증서가 사설일 경우 none 으로 설정하여 인증서 검사를 하지 않도록 함
#xpack.monitoring.elasticsearch.sniffing: false
#xpack.monitoring.collection.interval: 10s
#xpack.monitoring.collection.pipeline.details.enabled: true
내용생략....
-- 내용 수정 -----------------------------------------------------------------------------------
```

## Logstash .conf 파일 설정 및 실행
- 기존에 있던 Sample 에서 사용했던 부분을 예시로 제공함
- Elastcsearch 에 SSL 과 계정이 설정 되었으므로 해당 부분을 추가로 설정 해야 접속을 할 수 있음
```shell
# 작업 디렉토리 이동
logstash$ cd conf-file/sample1

# sample1.conf 파일 수정
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
    user => "elastic" <== 내용추가
    password => "changeme" <== 내용추가
    ssl => true <== 내용추가
    ssl_certificate_verification => false <== 내용추가
  }
  stdout {}
}
-- 내용 작성 -----------------------------------------------------------------------------------

# 실행
sample1$ ./start.sh
```

# Metricbeat X-Pack 적용
- Elasticsearch X-Pack 적용을 위해 미리 중지되었다고 가정함
- 복잡한 내용은 없으며, 설정 파일 수정 후 실행하면 완료
## Metricbeat 설정 파일 수정
```shell
# 작업 디렉토리 이동
~$ cd metricbeat

# metricbeat.yml 파일 수정
metricbeat$ sudo vi metricbeat.yml
-- 내용 수정 -----------------------------------------------------------------------------------
내용 생략....
# Starting with Beats version 6.0.0, the dashboards are loaded via the Kibana API.
# This requires a Kibana endpoint configuration.
setup.kibana:
  # Kibana Host
  # Scheme and port can be left out and will be set to the default (http and 5601)
  # In case you specify and additional path, the scheme is required: http://localhost:5601/path
  # IPv6 addresses should always be defined as: https://[2001:db8::1]:5601
  #host: "localhost:5601"
  host: "kibana_04:5601"
  username: "elastic" <== 내용 추가 및 수정
  password: "changeme" <== 내용 추가 및 수정
내용 생략....
#-------------------------- Elasticsearch output ------------------------------
output.elasticsearch:
  # Array of hosts to connect to.
  # hosts: ["localhost:9200"]
  hosts: ["elastic-01:9200", "elastic-02:9200", "elastic-03:9200"]
  # Optional protocol and basic auth credentials.
  protocol: "https" <== 주석 해제
  username: "elastic" <== 주석 해제
  password: "changeme" <== 주석 해제 및 내용 수정
  ssl.verification_mode: none <== 내용 추가
내용 생략....
-- 내용 수정 -----------------------------------------------------------------------------------
```

## Metricbeat 실행
```shell
# 실행
metricbeat$ ./start.sh
```

# 참고
- http://kimjmin.net/2018/08/2018-08-install-security-over-es63/
- https://www.elastic.co/guide/en/elastic-stack-overview/current/ssl-tls.html
- https://www.elastic.co/guide/en/kibana/current/settings.html
- https://www.elastic.co/guide/en/logstash/current/configuring-logstash.html
- https://www.elastic.co/guide/en/logstash/current/plugins-outputs-elasticsearch.html
- https://www.elastic.co/guide/en/beats/metricbeat/current/configuration-monitor.html
