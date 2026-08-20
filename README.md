# homeserver-iac

라즈베리파이 홈서버의 관측성(Observability) 스택, GitOps 배포 파이프라인, 인프라 설정을 코드로 관리하는 저장소.

## 아키텍처

\`\`\`
[외부(카페 등)]
    │ WireGuard 암호화 터널
    ▼
[라즈베리파이 5, wg0: 10.0.0.1]
    │
    ├── observability-stack (Docker Compose)
    │     ├── Prometheus  (지표 수집·평가)
    │     ├── node-exporter (호스트 CPU/메모리/디스크)
    │     ├── cAdvisor (컨테이너별 지표)
    │     ├── wireguard-exporter (VPN 핸드셰이크 상태)
    │     ├── Alertmanager (알림 라우팅 → Telegram)
    │     ├── Loki + Promtail (컨테이너 로그 수집·검색)
    │     └── Grafana (대시보드)
    │
    ├── k3s + ArgoCD (GitOps)
    │     └── k8s-manifests/ 경로를 ArgoCD가 직접 감시, git push만으로 배포
    │           └── 실제 운영 워크로드: infra-bot (텔레그램 인프라 조회 봇)
    │
    └── UFW / Fail2Ban / WireGuard (Ansible로 코드화)
\`\`\`

## 구성

- **observability-stack/** — Prometheus + Grafana + Alertmanager + cAdvisor + node-exporter + wireguard-exporter + Loki/Promtail을 Docker Compose로 배포. 알림 룰 5종(CPU/메모리/디스크 임계치, 서비스 다운, WireGuard 핸드셰이크 정체)과 컨테이너 로그 수집·검색까지 포함해 지표·알림·로그 세 축을 모두 커버.
- **ansible-homeserver/** — UFW·Fail2Ban·WireGuard 설정과 관측성 스택 배포를 Ansible 롤로 코드화. 비밀값(WireGuard 개인키, Grafana 비밀번호, Telegram 봇 토큰)은 Ansible Vault로 분리 관리.
- **k8s-manifests/** — k3s 클러스터에 ArgoCD로 배포하는 애플리케이션 매니페스트. `argocd-application.yaml`, `infra-bot-application.yaml`에 각각 저장소·경로를 감시하도록 등록해두면, 이후로는 해당 저장소에 push하는 것만으로 클러스터에 자동 반영됨(GitOps).

## 설계 결정 및 트러블슈팅

실제로 겪은 문제와 해결 과정. (환경: Raspberry Pi 5, Debian trixie, Docker 29, k3s)

- **UFW가 도커 컨테이너 간 통신까지 차단** — `network_mode: host`로 띄운 wireguard-exporter를 Prometheus 컨테이너가 스크레이프하려 하자 실패. 원인은 이 트래픽이 wg0도, 51820도 아닌 도커 브리지 네트워크(`172.18.0.0/16`)에서 들어와서 UFW 기본 정책(전부 차단)에 걸린 것. 해당 서브넷만 targeted로 허용하는 규칙을 추가해 해결.
- **Docker 이미지 자체의 기본 커맨드가 깨져 있었음** — `mindflavor/prometheus-wireguard-exporter` 최신 이미지의 CLI 파서가 업데이트되면서, 이미지에 내장된 기본 `CMD`가 무효한 값이 되어 있었음. `docker inspect`로 실제 사용 중인 Cmd를 확인해 원인을 좁히고, compose에서 유효한 값으로 완전히 덮어써서 해결.
- **Ansible의 거짓 변경(false diff) 방지** — WireGuard 설정을 템플릿화할 때 키 순서가 원본과 달라 매 실행마다 "변경됨"으로 잡혀 불필요하게 서비스가 재시작되는 문제 발견. 템플릿의 키 순서를 원본과 일치시켜 진짜 멱등성을 확보.
- **dry-run이 실제 장애를 사전에 차단** — 변수 파일의 오타(`10.8.0.1/24`)로 인해 실제 적용 시 WireGuard 서버 주소가 바뀌면서 VPN이 끊길 뻔한 상황을 `--check --diff`로 사전에 발견. 원격 서버 설정 변경은 실제 적용 전 반드시 dry-run으로 diff를 확인하는 것을 원칙으로 삼음.
- **비밀값과 코드의 분리** — 최초 설계에서는 로컬 관측성 스택 폴더를 그대로 서버에 복사하는 방식이라, 서버에서 직접 수정한 Telegram 봇 토큰이 재배포 시 되돌아갈 위험이 있었음. 해당 값을 Ansible Vault 관리 변수로 승격하고 별도 템플릿 태스크로 분리해, 배포와 비밀 관리를 동시에 안전하게 만듦.
- **Fail2Ban 설정을 파일 전체가 아닌 변경분만 관리** — 실제 `jail.local`은 Debian 기본 설정 전체를 복사한 파일이었음. 파일 전체를 템플릿으로 관리하는 대신 실제로 커스터마이징된 값(`ignoreip`, `sshd` 잡의 재시도/차단 시간)만 `ini_file` 모듈로 targeted 관리해, 유지보수 범위를 최소화.
- **로그 검증 대상을 잘못 고른 경험** — 로그 파이프라인(Loki/Promtail) 검증 시 알림 처리를 담당하는 Alertmanager 로그로 확인하려 했으나, Alertmanager는 알림을 정상적으로 처리·전송할 때 별도 로그를 남기지 않는 것으로 확인됨(에러 시에만 로그 기록). Promtail 자체 로그(`added Docker target` 정상 등록)와, 재시작 시 반드시 로그를 남기는 node-exporter로 대상을 바꿔 재현해 파이프라인 정상 동작을 최종 검증함 — 검증 대상 선정도 관측 대상의 실제 로깅 동작을 먼저 파악해야 한다는 교훈.
- **업스트림 커널 버그 발견 및 우회 판단** — 라즈베리파이 5 + 커널 6.12 조합에서 k3s 설치 시 memory cgroup이 활성화되지 않는 문제 발생. 원인은 디바이스트리(DTB)에 `cgroup_disable=memory`가 하드코딩되어 있고, 이를 취소하던 `cgroup_enable` 커널 로직이 6.12부터 빠진 것 — 라즈베리파이 공식 커널 저장소에 이미 보고된 이슈([raspberrypi/linux#6980](https://github.com/raspberrypi/linux/issues/6980))였으나 PR 없이 closed된, 사실상 미해결 상태였음. 커널 다운그레이드 등 근본 수정 대신, 실제 클러스터 동작에 미치는 영향(파드별 메모리 cgroup 제한만 비활성화, 클러스터 자체는 정상 동작)을 먼저 검증하고, 목표(GitOps 데모) 대비 리스크가 낮은 우회를 선택함.
- **GitOps 핵심 동작 실증** — ArgoCD의 `selfHeal`(수동으로 replica 수를 변경해도 git 상태로 자동 원복)과 git revert 기반 롤백(잘못된 이미지 태그로 배포 사고를 낸 뒤 `git revert`만으로 자동 복구)을 데모 앱으로 먼저 검증. 이후 실제 운영 서비스(infra-bot)를 같은 파이프라인에 배포해, 데모가 아닌 실제 워크로드에서도 동일하게 동작함을 확인.
- **네임스페이스를 넘나드는 트래픽이 UFW에 막히는 패턴 재확인** — k3s 파드에서 Docker Compose로 띄운 Prometheus(`10.0.0.1:9090`)로 나가는 트래픽이, wg0 인터페이스를 거치지 않는다는 이유로 기존 UFW 규칙에 걸려 차단됨. 앞서 겪은 wireguard-exporter 사례와 같은 근본 패턴(컨테이너/파드 간 트래픽은 예상한 인터페이스를 안 거칠 수 있음)임을 인지하고, k3s 파드 네트워크 대역을 명시적으로 허용하는 규칙을 추가해 해결(자세한 내용은 infra-bot 저장소 README 참고).

## 사용법

### 1. observability-stack

\`\`\`bash
cp observability-stack/.env.example observability-stack/.env
# .env에 WG_IP, GRAFANA_ADMIN_PASSWORD 채운 뒤
scp -r observability-stack <host>:~/
ssh <host> "cd observability-stack && docker compose up -d"
\`\`\`

### 2. ansible-homeserver

\`\`\`bash
cd ansible-homeserver
ansible-galaxy collection install -r requirements.yml
cp group_vars/homeserver/vault.yml.example group_vars/homeserver/vault.yml
# vault.yml에 실제 비밀값 채운 뒤
ansible-vault encrypt group_vars/homeserver/vault.yml

# 반드시 dry-run 먼저
ansible-playbook -i inventory.ini playbook.yml --ask-vault-pass --check --diff

# 확인 후 실제 적용
ansible-playbook -i inventory.ini playbook.yml --ask-vault-pass
\`\`\`

### 3. k3s + ArgoCD (GitOps)

\`\`\`bash
# Pi에서 — 최초 1회만
curl -sfL https://get.k3s.io | sh -
# cgroup_disable=memory 관련 이슈가 있으면 위 트러블슈팅 항목 참고

kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
kubectl apply -f k8s-manifests/argocd-application.yaml
kubectl apply -f k8s-manifests/infra-bot-application.yaml
\`\`\`

이후로는 각 앱 저장소(`k8s-manifests/` 경로)에 push하는 것만으로 ArgoCD가 자동 동기화함(기본 3분 주기, 즉시 반영하려면 `kubectl patch application <name> -n argocd --type merge -p '{"operation":{"sync":{}}}'`).

## 스택

Docker Compose · Prometheus · Grafana · Alertmanager · cAdvisor · Loki · Promtail · Ansible · Ansible Vault · WireGuard · UFW · Fail2Ban · k3s · ArgoCD