# homeserver-iac

## 1. observability-stack (Docker Compose)

1. `.env.example`을 `.env`로 복사하고 WG_IP(파이의 wg0 인터페이스 IP)와 GRAFANA_ADMIN_PASSWORD를 채운다.
2. 파이에 이 폴더를 올린다 (scp 또는 git).
3. `cd observability-stack && docker compose up -d`
4. `alertmanager/alertmanager.yml`의 bot_token, chat_id를 실제 텔레그램 봇 정보로 채운 뒤 `docker compose restart alertmanager`.
5. Grafana(`http://<WG_IP>:3000`) 접속 → Data source에 Prometheus(`http://prometheus:9090`) 추가 → 대시보드 임포트(Node Exporter Full: 1860, cAdvisor: 893).

## 2. ansible-homeserver (설정 코드화)

1. `ansible-galaxy collection install -r requirements.yml`
2. `inventory.ini`에 파이 IP/사용자명 채움.
3. `group_vars/homeserver/vars.yml`을 실제 WireGuard 피어 정보로 채움.
4. `group_vars/homeserver/vault.yml.example`을 `vault.yml`로 복사 후 실제 비밀값 채우고 암호화:
   `ansible-vault encrypt group_vars/homeserver/vault.yml`
5. 실행: `ansible-playbook -i inventory.ini playbook.yml --ask-vault-pass`

## 주의

- 이 레포를 git에 올릴 경우 `vault.yml`(암호화된 파일)만 커밋하고, `vault.yml.example`의 평문 값이나 `.env`는 절대 커밋하지 않는다.
- 처음 실행 전에 지금 파이에 있는 실제 WireGuard 설정(`/etc/wireguard/wg0.conf`)과 fail2ban 설정(`/etc/fail2ban/jail.local`)을 먼저 백업해둔다.
