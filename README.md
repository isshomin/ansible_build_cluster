# 🕸️ Ansible 기반 빌드 클러스터 (Amazon EC2 + 로컬 VM)


## ✒ 목적

원래는 CI/CD 하면 Jenkins만 떠올렸지만, **GitHub Actions와 Ansible을 조합해도 충분히 자동화가 가능하다는 점**이 흥미로웠습니다. </br>
특히 빌드 자체는 Amazon EC2와 VirtualBox VM에 설치한 Self-hosted Runner로 처리하고, 배포는 Ansible이 맡는 구조를 직접 구성해보고 싶었습니다. </br>
단순히 빌드 시간을 줄이는 것보다 **리소스 분리, 안정성, 병렬성**을 확보할 수 있는 빌드 클러스터의 설계와 장애 대응 테스트에 초점을 두었으며, 클라우드와 로컬을 혼합해 **실제 서비스 환경과 유사성을 확보하면서도 비용 효율까지 고려한 구성**을 만들고자 했습니다. </br>

</br>

---

## ⚙ 초기설정
### 🏗️ 빌드 클러스터 (Controller + Runners)

<img width="806" height="352" alt="image" src="https://github.com/user-attachments/assets/90626e06-ecd5-45b9-a2c0-251e1cf6cb5f" />


**Controller (VirtualBox VM)** / IP: 10.0.2.72 </br>
- Ansible + GitHub Actions Orchestrator
- GitHub Actions에서 deploy job 실행
- Ansible Playbook 실행을 통해 배포 자동화

**EC2 Runner (Public Subnet)** / Public IP: 13.xxx.xxx.xxx </br>
- GitHub Actions Self-hosted Runner (기본 빌드 수행)

**Local Runner (VirtualBox VM)** / IP: 10.0.2.73 </br>
- GitHub Actions Self-hosted Runner (fallback 용도)

</br>

### 🚪 Bastion Host (Runner EC2 겸용)
**EC2 Runner (Public Subnet)** / Public IP: 13.xxx.xxx.xxx </br>
- Private Subnet 접근을 위한 SSH ProxyJump
- 비용 문제로 Runner EC2가 Bastion 역할까지 겸하도록 설계

</br>

### 📦 배포 대상 서버 (Targets)

**EC2 Deploy (Private Subnet EC2)** / IP: 10.10.2.20 </br>
- 서비스 배포용 서버 (Ansible이 Bastion 경유 후 접속)
- 실제 클라우드 배포 환경

**Local Deploy (VirtualBox VM)** / IP: 10.0.2.74 </br>
- 로컬 테스트/검증용 배포 서버
- 단계적·선택적 배포 전략 실습을 위한 환경

</br>

### 📄 Ansible Inventory(.ini)
```ini
# ---------------- 빌드 러너 ----------------
[local_runners]
local-runner ansible_host=10.0.2.73 ansible_user=xxx ansible_port=2222 ansible_ssh_private_key_file=/home/xxx/ansible/ssh_keys/id_rsa

[aws_runners]
ec2-runner ansible_host=13.xxx.xxx.xxx ansible_user=ubuntu ansible_port=2222 ansible_ssh_private_key_file=/home/xxx/ansible/ssh_keys/local-key.pem

[all_runners:children]
local_runners
aws_runners

# ---------------- 배포 타깃 ----------------
[local_deploy]
local-deploy ansible_host=10.0.2.74 ansible_user=xxx ansible_port=2222 ansible_ssh_private_key_file=/home/xxx/ansible/ssh_keys/id_rsa

[aws_deploy]
aws-deploy ansible_host=10.10.2.20 ansible_user=ubuntu ansible_ssh_private_key_file=/home/xxx/ansible/ssh_keys/local-key.pem ansible_ssh_common_args='-o ProxyJump=ubuntu@13.xxx.xxx.xxx:2222' # Bastion Host 경유

[all_deploy:children]
local_deploy
aws_deploy

```
- 빌드/배포 대상 서버 정보를 정의하는 설정 파일

</br>

### 🌐 VPC 구성도
<img width="1668" height="752" alt="image" src="https://github.com/user-attachments/assets/beec408b-3e46-429a-8835-c5083f9b78d3" />

### 🖥️ EC2
<img width="1266" height="213" alt="image" src="https://github.com/user-attachments/assets/65e92733-ba66-462c-847e-4370863b53e1" />


</br>

---

## 🏗️ Architecture

![image](https://github.com/user-attachments/assets/12a09935-579e-4750-a0dc-9a779e9af561)

</br>

---

## 🔹 GitHub Actions Workflow Jobs

<img width="1374" height="418" alt="image" src="https://github.com/user-attachments/assets/52fa4680-cbd1-4d94-a160-f29094a7fc1b" />

### 📂 build-ec2
- Gradle 빌드 + Docker 이미지 빌드 & 푸시
- 기본 빌드 경로 (우선 실행 대상)

### 📂 build-local
- Gradle 빌드 + Docker 이미지 빌드 & 푸시
- EC2 Runner 실패 시 fallback 용도

### 📂 promote-latest
- EC2 빌드 성공 시 → latest 태그로 승격
- EC2 빌드 실패 시 → Local 빌드 fallback 승격

### 📂 deploy
- Ansible Playbook 실행
- 입력값(aws, local, all)에 따라 배포 타깃 선택

</br>

---

## 🔸 GitHub Actions Workflow 시나리오

### ✅ 시나리오 1: EC2 빌드 성공

<img width="1374" height="418" alt="image" src="https://github.com/user-attachments/assets/52fa4680-cbd1-4d94-a160-f29094a7fc1b" />
</br>
<img width="1547" height="418" alt="image" src="https://github.com/user-attachments/assets/1b3c774f-6323-4f06-8b05-3acc8e1d20ff" />

- ec2-**** 이미지가 정상 빌드 및 푸시됨

- latest 태그와 Digest가 동일 → EC2 빌드 결과가 최종본으로 승격

- 기본 경로(EC2 Runner)가 정상적으로 동작한 케이스

### ⚠️ 시나리오 2: EC2 빌드 실패 → Local Runner Fallback

<img width="1374" height="409" alt="image" src="https://github.com/user-attachments/assets/876b39f2-e7b1-42e3-a7fc-d7a55bb3864d" />
</br>
<img width="1538" height="423" alt="image" src="https://github.com/user-attachments/assets/ed8cd8fd-a360-4150-a158-8cd27cad138f" />

- EC2 Runner에서 오류/중단 발생

- local-**** 이미지가 latest로 승격 → 로컬 빌드가 최종본으로 대체

- 장애 상황에서도 빌드 파이프라인이 끊기지 않고 지속됨

</br>

---

## ✅ 배포 완료

### 📄 Deploy.yml
```yml
---
- name: Deploy Spring Boot App with Docker
  hosts: all_deploy
  become: true

  vars:
    app_name: build-cluster
    image_name: your_dockerhub_user/build-cluster:latest
    container_port: 8085
    host_port: 8085

  tasks:
    - name: Ensure Docker is installed
      apt:
        name: docker.io
        state: present
        update_cache: yes
      tags: docker

    - name: Ensure Docker service is running
      service:
        name: docker
        state: started
        enabled: yes
      tags: docker

    - name: Pull latest image from Docker Hub
      community.docker.docker_image:
        name: "{{ image_name }}"
        source: pull

    - name: Remove old container if exists
      community.docker.docker_container:
        name: "{{ app_name }}"
        state: absent

    - name: Run container with new image
      community.docker.docker_container:
        name: "{{ app_name }}"
        image: "{{ image_name }}"
        state: started
        restart_policy: always
        ports:
          - "{{ host_port }}:{{ container_port }}"

    - name: Wait for app to be ready (max 30s)
      uri:
        url: "http://localhost:{{ host_port }}/actuator/health"
        method: GET
        status_code: 200
      register: healthcheck
      retries: 6
      delay: 5
      until: healthcheck.status == 200
      ignore_errors: yes

    - name: Show last 20 lines of container logs
      command: docker logs --tail 20 "{{ app_name }}"
      register: app_logs
      ignore_errors: true

    - debug:
        var: app_logs.stdout_lines
```

### 🗄️ EC2 (Private Subnet)

<img width="698" height="245" alt="image" src="https://github.com/user-attachments/assets/681150f2-7189-4505-a903-9180a55b4678" /> </br>

- 브라우저 `localhost:18085` 접속 성공

### 🗄️ VirtualBox VM (Local)

<img width="1059" height="226" alt="image" src="https://github.com/user-attachments/assets/207be16e-6969-4f6a-bd8d-b11b6d767ca2" /> </br>

- 브라우저 `localhost:8085` 접속 성공

</br>

---

## 💥 트러블슈팅


</br>

---

## 💭 회고

