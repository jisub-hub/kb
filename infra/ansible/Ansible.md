---
tags:
  - infra
  - ansible
  - automation
  - devops
created: 2026-06-15
---

# Ansible

> [!summary] 한 줄 요약
> **에이전트 없이** SSH만으로 서버 설정·패키지 설치·파일 배포를 자동화하는 도구. Python으로 구현된 YAML Playbook을 실행해 인프라를 **멱등성** 있게 관리한다.

---

## 1. 핵심 개념

| 개념 | 설명 |
|------|------|
| **Inventory** | 관리할 호스트 목록 (IP, 그룹 정의) |
| **Playbook** | 자동화 작업을 순서대로 기술한 YAML |
| **Play** | 특정 호스트 그룹에 적용할 Task 묶음 |
| **Task** | 단일 작업 단위 (모듈 한 번 호출) |
| **Module** | 실제 작업을 수행하는 기능 단위 (apt, copy, service …) |
| **Handler** | 특정 Task 완료 시 딱 한 번 실행되는 Task (예: 서비스 재시작) |
| **Role** | Playbook 재사용 단위 (폴더 구조 규약) |
| **Variable** | Playbook 내 동적 값 |

---

## 2. Inventory

```ini
# inventory/hosts.ini
[webservers]
web1 ansible_host=10.0.1.10
web2 ansible_host=10.0.1.11

[dbservers]
db1  ansible_host=10.0.2.10

[all:vars]
ansible_user=ubuntu
ansible_ssh_private_key_file=~/.ssh/id_rsa
ansible_python_interpreter=/usr/bin/python3
```

```yaml
# inventory/hosts.yml (YAML 형식)
all:
  children:
    webservers:
      hosts:
        web1: { ansible_host: 10.0.1.10 }
        web2: { ansible_host: 10.0.1.11 }
    dbservers:
      hosts:
        db1: { ansible_host: 10.0.2.10 }
```

---

## 3. Playbook 기본 구조

```yaml
# site.yml
---
- name: 웹 서버 설정
  hosts: webservers
  become: true           # sudo 권한으로 실행
  vars:
    app_port: 8080
    app_version: "1.2.0"

  tasks:
    - name: 패키지 업데이트
      apt:
        update_cache: true
        cache_valid_time: 3600

    - name: Java 설치
      apt:
        name: openjdk-21-jre-headless
        state: present

    - name: 앱 JAR 배포
      copy:
        src: build/libs/app.jar
        dest: /opt/app/app.jar
        owner: appuser
        mode: "0755"
      notify: restart app   # 변경 시 핸들러 트리거

    - name: 서비스 활성화
      systemd:
        name: myapp
        enabled: true
        state: started

  handlers:
    - name: restart app
      systemd:
        name: myapp
        state: restarted
```

---

## 4. 자주 쓰는 모듈

```yaml
# 파일·디렉토리
- copy:
    src: files/nginx.conf
    dest: /etc/nginx/nginx.conf
    owner: root
    mode: "0644"

- template:              # Jinja2 템플릿 → 파일 생성
    src: templates/app.conf.j2
    dest: /etc/app/app.conf

- file:
    path: /opt/app/logs
    state: directory     # directory | file | absent | link
    owner: appuser
    mode: "0755"

# 패키지
- apt:                   # Ubuntu/Debian
    name: [nginx, curl]
    state: present        # present | absent | latest

- yum:                   # RHEL/CentOS
    name: nginx
    state: present

# 서비스
- systemd:
    name: nginx
    state: started        # started | stopped | restarted | reloaded
    enabled: true

# 명령 실행 (멱등성 없음 → 가능하면 전용 모듈 사용)
- command: /opt/app/migrate.sh
  args:
    creates: /opt/app/.migrated   # 파일 있으면 skip → 멱등성 확보

- shell: ps aux | grep myapp

# 사용자
- user:
    name: appuser
    shell: /bin/bash
    groups: docker
    append: true
```

---

## 5. 변수 (Variables)

```yaml
# 변수 우선순위 (높을수록 우선)
# extra vars (-e) > task vars > block vars > play vars > host_vars > group_vars > role defaults

# group_vars/webservers.yml
app_port: 8080
java_opts: "-Xmx512m"

# host_vars/web1.yml
app_port: 9090   # web1만 다른 포트
```

```yaml
# 변수 사용 — Jinja2 {{ }}
- name: 앱 시작
  command: java {{ java_opts }} -jar /opt/app/app.jar --server.port={{ app_port }}

# 조건부 실행
- name: Ubuntu에서만 실행
  apt:
    name: nginx
    state: present
  when: ansible_distribution == "Ubuntu"

# 반복
- name: 디렉토리 생성
  file:
    path: "{{ item }}"
    state: directory
  loop:
    - /opt/app
    - /opt/app/logs
    - /opt/app/config
```

---

## 6. Jinja2 템플릿

```jinja2
{# templates/app.conf.j2 #}
server.port={{ app_port }}
spring.profiles.active={{ env }}
logging.level.root={{ log_level | default('INFO') }}

{% if enable_ssl %}
server.ssl.enabled=true
server.ssl.key-store={{ ssl_keystore }}
{% endif %}
```

---

## 7. Role 구조 — 재사용 단위

```
roles/
└── java-app/
    ├── tasks/
    │   └── main.yml      # 주 태스크 목록
    ├── handlers/
    │   └── main.yml      # 핸들러
    ├── templates/
    │   └── app.conf.j2   # Jinja2 템플릿
    ├── files/
    │   └── app.service   # 정적 파일
    ├── vars/
    │   └── main.yml      # 고정 변수 (재정의 불가에 가까움)
    └── defaults/
        └── main.yml      # 기본 변수 (재정의 가능)
```

```yaml
# Playbook에서 Role 사용
- hosts: webservers
  roles:
    - role: java-app
      vars:
        app_port: 8080
```

---

## 8. 주요 CLI 명령어

```bash
# 연결 확인
ansible all -i inventory/hosts.ini -m ping

# 특정 그룹만
ansible webservers -i inventory/ -m shell -a "uptime"

# Playbook 실행
ansible-playbook -i inventory/ site.yml
ansible-playbook -i inventory/ site.yml --limit web1   # 특정 호스트만
ansible-playbook -i inventory/ site.yml --tags deploy  # 특정 태그만
ansible-playbook -i inventory/ site.yml --check        # dry-run (변경 없이 예측)
ansible-playbook -i inventory/ site.yml -v             # verbose (-vvv 더 상세)

# 변수 오버라이드
ansible-playbook site.yml -e "app_version=2.0.0 env=prod"

# 암호화 (Vault)
ansible-vault encrypt secrets.yml
ansible-playbook site.yml --ask-vault-pass
```

---

## 9. Ansible Vault — 시크릿 관리

```bash
ansible-vault create secrets.yml       # 새 암호화 파일
ansible-vault edit secrets.yml         # 수정
ansible-vault view secrets.yml         # 조회
```

```yaml
# secrets.yml (암호화된 상태로 저장)
db_password: "prod_secret_pw"
api_key: "abcdef123456"
```

---

## 10. 베스트 프랙티스

- **멱등성 확보** — 몇 번 실행해도 결과가 같도록. `command/shell` 최소화
- **become은 필요한 Task에만** — play 전체 대신 task 단위 권한 상승
- **변수는 group_vars/host_vars 분리** — Playbook에 하드코딩 금지
- **시크릿은 Vault** — 평문 패스워드 코드에 포함 ❌
- **check mode (dry-run)** — 프로덕션 적용 전 항상 확인
- **tag 활용** — `--tags deploy`, `--tags config` 로 부분 실행 가능하게

---

## 관련
- [[Terraform]] · [[Kubernetes]] · [[infra/_index]]
