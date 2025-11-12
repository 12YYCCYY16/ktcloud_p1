-----

# ☁️ OpenStack Node Auto-Scaling System (Prometheus, Alertmanager, Jenkins) 통합 가이드

본 프로젝트는 **Prometheus**와 **Alertmanager**를 활용하여 OpenStack 인스턴스의 CPU 사용량을 모니터링하고, 특정 임계치 초과 시 **Jenkins 웹훅**을 트리거하여 자동 스케일링 또는 복구 조치를 수행하는 시스템을 구축합니다.

-----

## 🌟 프로젝트 컴포넌트

| 컴포넌트 | 역할 | 포트 |
| :--- | :--- | :--- |
| **Prometheus** | 메트릭 수집 및 경고 규칙 평가 | 9090 |
| **Alertmanager** | 경고 라우팅 및 Jenkins Webhook 전송 | 9093 |
| **Grafana** | 수집된 메트릭 시각화 (대시보드) | 3000 |
| **Node Exporter** | OpenStack 노드의 시스템 메트릭 수집 | 9100 |
| **Jenkins** | 자동 조치(`AutoHealingJob`) 실행 | 8080 |

-----

## 🎯 1단계: 필수 파일 및 환경 준비

### 1\. 파일 전송 (Prometheus 서버)

Prometheus 서버로 사용할 팀원분의 컴퓨터에 다음 4개의 파일을 **동일한 디렉토리에 배치**합니다.

  * `docker-compose.yml`
  * `prometheus.yml`
  * `alertmanager.yml`
  * `alerts.yml`

### 2\. Jenkins 설정 확인

자동 조치(Auto-Scaling)를 실행할 Jenkins 서버에 \*\*`AutoHealingJob`\*\*이 생성되어 실행 중인지 확인합니다.

> 이 Job의 웹훅 토큰(`YOUR_TOKEN`)이 `alertmanager.yml` 파일 내 설정과 일치하는지 확인해야 합니다.

-----

## 💻 2단계: OpenStack 모니터링 대상 설정 (모든 노드에서 수행)

Prometheus가 메트릭을 수집할 수 있도록 **모든 OpenStack 노드**에서 다음 작업을 수행해야 합니다.

### 1\. Node Exporter 설치 및 실행

모니터링 대상이 되는 \*\*모든 OpenStack 인스턴스(노드)\*\*에 접속하여 Node Exporter를 설치하고 실행합니다.

```bash
# [모든 OpenStack 노드에서 실행]

# 1. Node Exporter 다운로드 (예시 버전: v1.7.0)
wget https://github.com/prometheus/node_exporter/releases/download/v1.7.0/node_exporter-1.7.0.linux-amd64.tar.gz

# 2. 압축 해제 및 실행 파일 이동
tar xvfz node_exporter-1.7.0.linux-amd64.tar.gz
sudo mv node_exporter-1.7.0.linux-amd64/node_exporter /usr/local/bin/

# 3. Node Exporter 실행 (백그라운드 실행 권장)
/usr/local/bin/node_exporter &

# 4. 9100 포트로 메트릭 노출 확인
curl http://localhost:9100/metrics
```

### 2\. 방화벽 및 보안 그룹 설정 (9100 포트 허용)

Prometheus 서버가 각 노드의 9100 포트로 접근할 수 있도록 네트워크 설정을 완료해야 합니다.

  * **각 OpenStack 노드 내부 방화벽 (firewalld/ufw):** **9100 포트의 TCP 접근**을 허용합니다.
  * **OpenStack 보안 그룹 (Security Group):**
      * **Prometheus 서버의 IP 주소** 또는 모니터링 네트워크 대역 전체에 대해 **인바운드(Inbound) 9100/TCP 접근**을 허용하는 규칙을 모니터링 대상 인스턴스에 적용된 보안 그룹에 추가합니다.

-----

## 📝 3단계: 구성 파일 수정 (Prometheus 서버에서)

Prometheus 서버 역할을 할 컴퓨터로 돌아와, 두 파일을 정확하게 수정합니다.

### 1\. `prometheus.yml` 수정: 노드 리스트 정의

`job_name`을 명확하게 변경하고, Node Exporter가 설치된 **모든 OpenStack 노드의 IP 주소**를 `targets`에 추가합니다.

```yaml
# prometheus.yml 파일 내용 중 scrape_configs 섹션
scrape_configs:
...
  # 💡 [필수 수정 A] 잡 이름 변경 및 모든 노드 IP 추가
  - job_name: 'node_exporter_openstack_nodes' # 이름을 변경했습니다.
    static_configs:
      - targets:
          - '<Prometheus_Server_IP>:9100' # (옵션) Prometheus 서버가 자신을 모니터링할 경우
          - '192.168.1.10:9100'          # 👈 OpenStack 노드 1의 실제 IP
          - '192.168.1.11:9100'          # 👈 OpenStack 노드 2의 실제 IP
          # 테스트할 모든 OpenStack 노드를 여기에 추가해야 합니다.
```

### 2\. `alertmanager.yml` 수정: Jenkins URL 정의

Jenkins 웹훅 URL의 \*\*`<JENKINS_IP>`\*\*를 **Jenkins 서버의 실제 IP 주소**로 변경합니다.

```yaml
# alertmanager.yml 파일 내용 중 receivers 섹션
receivers:
...
  - name: 'jenkins-autoscale'
    webhook_configs:
      # 💡 [필수 수정 B] <JENKINS_IP>를 Jenkins 서버의 실제 IP로 변경
      - url: 'http://<JENKINS_IP>:8080/job/AutoHealingJob/buildWithParameters?token=YOUR_TOKEN'
        send_resolved: true
```

-----

## 🚀 4단계: 시스템 실행 및 최종 검증

### 1\. 서비스 실행

수정된 파일이 있는 디렉토리에서 Docker Compose를 실행합니다.

```bash
docker-compose up -d
```

### 2\. Prometheus 타겟 상태 확인 (성공의 척도)

1.  웹 브라우저로 \*\*`http://<Prometheus_IP>:9090`\*\*에 접속합니다.
2.  상단 메뉴에서 **Status** → **Targets**를 클릭합니다.
3.  `node_exporter_openstack_nodes` 잡 아래에 리스트업된 **모든 OpenStack 노드**가 **`State: UP`** 인지 확인합니다.
    > ⚠️ **주의:** 상태가 **DOWN**이라면, 해당 노드의 방화벽(9100 포트)이나 IP 주소 설정을 **2단계**에서 다시 확인해야 합니다.

### 3\. 자동 조치 기능 테스트

1.  **OpenStack 노드 부하 발생:** **UP** 상태인 노드 중 하나에 접속하여 CPU 사용률을 **80% 이상**으로 5분 이상 유지합니다.
    ```bash
    # (예시: CPU 코어 수만큼 실행)
    yes > /dev/null &
    ```
2.  **Jenkins 웹훅 확인:** **Jenkins 서버**에 접속하여 `AutoHealingJob`의 빌드 히스토리를 확인합니다. **경고가 발생한 직후 자동으로 빌드가 시작**되었다면 통합 성공입니다.
3.  **부하 제거:** 테스트 완료 후에는 `killall yes` 명령으로 부하를 제거합니다.
