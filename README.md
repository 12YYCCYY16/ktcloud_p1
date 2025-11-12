☁️ OpenStack Node Auto-Scaling System (Prometheus, Alertmanager, Jenkins)

🌟 프로젝트 소개
본 프로젝트는 Prometheus와 Alertmanager를 활용하여 OpenStack 인스턴스의 CPU 사용량을 모니터링하고, 특정 임계치 초과 시 Jenkins 웹훅을 트리거하여 자동 스케일링 또는 복구 조치를 수행하는 시스템을 구축합니다.


컴포넌트,역할,포트
Prometheus,메트릭 수집 및 경고 규칙 평가,9090
Alertmanager,경고 라우팅 및 Jenkins Webhook 전송,9093
Grafana,수집된 메트릭 시각화 (대시보드),3000
Node Exporter,OpenStack 노드의 시스템 메트릭 수집,9100
Jenkins,자동 조치(AutoHealingJob) 실행,8080

⚙️ 설정 파일 및 환경 구성
1. 사전 준비 사항
OpenStack 노드: 모니터링 대상이 되는 모든 OpenStack 인스턴스에 **node_exporter**를 설치하고 9100 포트가 Prometheus 서버에서 접근 가능하도록 설정해야 합니다.

Jenkins 서버: 자동 스케일링 조치를 위한 **AutoHealingJob**이 생성되어 실행 중이어야 합니다.

