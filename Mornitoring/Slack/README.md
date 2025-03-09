# 📊 Grafana - Slack 연동

| 현재 서버 상태를 수동으로 점검하는 방식이어서 문제 발생 시 즉각적인 대응이 어려운 상황입니다.
이를 해결하기 위해 Grafana와 Slack을 연동하여 문제 발생시 자동으로 알림을 받을 수 있도록 설정하였습니다. |
| --- |

### **🔍 어떤 조건에서 알림을 받을 것인지?**

- **인프라팀**
    - CPU, 메모리, 디스크 사용량이 **임계치를 초과**하면 알림을 전송
    - 서버에서 **비정상적인 상태 감지 시** Slack으로 즉시 알림
- **개발팀**
    - 로그에서 `ERROR`, `CRITICAL` 등의 **특정 패턴이 감지되면** Slack으로 알림 전송
    - **빠른 오류 파악**을 통해 문제 해결 시간을 단축

### **🎯 기대 효과**

✅ 서버 장애 및 성능 저하 발생 시 **즉각적인 대응 가능**

✅ **문제 발생 원인 파악 시간 단축** → 개발팀이 신속하게 수정 가능

✅ 반복적인 장애 패턴을 분석하여 **장기적인 안정성 향상**

## 🚀 1. Slack App 생성

Slack과 연동하려면 먼저 Slack에서 새로운 앱을 생성해야 합니다.

1. [Slack API 페이지](https://api.slack.com/apps)로 이동합니다.

![Image](https://github.com/user-attachments/assets/77474eb4-2ef4-4bbe-863b-6f38c71c1325)

1. `From scratch`를 클릭하여 새 앱을 생성합니다.
2. 앱 이름을 설정하고 사용할 워크스페이스를 선택한 후 앱을 생성합니다.

![Image](https://github.com/user-attachments/assets/961fd949-28d4-4ce7-bd1c-a12a9abb1f5f)

![Image](https://github.com/user-attachments/assets/5b856abe-90a4-4fe7-a368-051b7a979e45)

App 생성 완료

### ✅ Oauth & Permissions에서 Token 생성

1. `OAuth & Permissions` 메뉴로 이동합니다.
2. `Install to Workspace` 버튼을 클릭하고 권한을 허용하여 Token을 생성합니다.

![Image](https://github.com/user-attachments/assets/f7b78950-a011-487c-bd3e-afef948f5491)

- Install to Workspace를 클릭하고 허용을 눌러 Token 생성

![Image](https://github.com/user-attachments/assets/8e64f393-3ed9-49a5-82a0-d37f86a0339a)

![Image](https://github.com/user-attachments/assets/0705536a-235d-4c6e-af5a-a08013bf15bb)

### ✅ Incoming Webhooks 활성화

1. 왼쪽 메뉴에서 `Incoming Webhooks`을 클릭합니다.
2. `Incoming Webhooks` 기능을 활성화합니다.
3. `Add Webhook`을 클릭하여 Webhook URL을 추가합니다.

### 

![Image](https://github.com/user-attachments/assets/ff25a3f4-69dc-47a2-bbd3-f6bf005c6008)

![Image](https://github.com/user-attachments/assets/a3f9e74e-0d2e-4e2c-a761-b19532fe9132)

- Add WebhookUrl

![Image](https://github.com/user-attachments/assets/16a2e5bf-a728-4195-9f1a-fe5b43a7b767)

### ✅ Slack에 @grafana 추가

- Slack에서 `@grafana` 앱을 추가하여 알림을 받을 수 있도록 설정합니다.

![Image](https://github.com/user-attachments/assets/89a0142c-0ce1-4713-a128-75589963fe7d)

## 📈 2. Grafana 설정

### 🔔 Alert Rule 생성

1. Grafana에서 `Alerting` → `New Alert Rule`로 이동합니다.
2. 적절한 Rule name과 Query를 설정합니다.
3. 알림을 구분할 수 있도록 Folder 및 Group을 설정합니다.

![Image](https://github.com/user-attachments/assets/ed4b2155-ab4d-4390-b40e-ad066336535f)

![Image](https://github.com/user-attachments/assets/4b9813a7-6164-49ec-9a87-e133920584c9)

### 📩 Contact Point 설정

1. `Alerting` → `Contact points`로 이동합니다.
2. Slack Webhook URL을 입력하여 Slack으로 알림을 받을 수 있도록 설정합니다.

![Image](https://github.com/user-attachments/assets/f75505de-dd35-4751-8447-8f1d016bdbe8)

![Image](https://github.com/user-attachments/assets/3d972841-18d3-45f5-bc56-bd88e69d035e)

### 🔍 Contact 설정 확인

- 테스트 알림을 전송하여 Slack과의 연결이 정상적으로 이루어졌는지 확인합니다.

![Image](https://github.com/user-attachments/assets/6ba534ea-9dc3-47d3-8291-cc18f59e0e47)
