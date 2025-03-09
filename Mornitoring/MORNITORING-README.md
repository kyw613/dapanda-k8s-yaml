<img width="481" alt="Image" src="https://github.com/user-attachments/assets/ddc73c77-f5a7-46c7-aa00-9ac088c94bf9" />

🛠 **EC2 모니터링 서버**

| `Grafana` | 모니터링 및 데이터 시각화 |
| --- | --- |
| `Loki` | 로그 저장 및 조회 |
| `Jaeger` | 트레이싱 데이터 저장 및 분석 |

🛠 **EKS 환경**

| `OpenTelemetry` | 애플리케이션에서 로그 및 트레이싱 수집 |
| --- | --- |
| `CloudWatch Agent` | 메트릭을 `CloudWatch`로 전송 |
|  |  |

## **🔍 모니터링 데이터 흐름**



1️⃣ **Django & Spring 애플리케이션** → `OpenTelemetry`로 **로그 및 트레이싱 전송**

2️⃣ `OpenTelemetry`는 데이터를 각각 전달

- **로그(log)** → `Loki`
- **트레이싱(tracing)** → `Jaeger`

3️⃣ `Grafana`를 통해 `Loki`, `Jaeger`, `CloudWatch` 데이터를 시각화하여 확인

## **🚀 Trouble Shooting**



✅ 원래는 `Prometheus` + `Node Exporter` 조합을 사용하려 했음

- 오픈소스 기반이라 **비용 절감** 가능
- 하지만 **Pull 방식**이라 `Node Exporter`에서 데이터를 가져올 수 없었음
- `Pushgateway`를 사용해도 문제 해결이 안 됨

✅ 결국 **완전 관리형 서비스인 CloudWatch**를 활용하여 문제 해결!

### Istio 서비스 메시에 OpenTelemetry를 확장 제공자로 통합

```c
apiVersion: install.istio.io/v1alpha1
kind: IstioOperator
spec:
  meshConfig:
    extensionProviders:
      - name: otel
        envoyOtelAls:
          service: opentelemetry-collector.istio-system.svc.cluster.local
          port: 4317
          logFormat:
            labels:
              pod: "%ENVIRONMENT(POD_NAME)%"
              namespace: "%ENVIRONMENT(POD_NAMESPACE)%"
              cluster: "%ENVIRONMENT(ISTIO_META_CLUSTER_ID)%"
              mesh: "%ENVIRONMENT(ISTIO_META_MESH_ID)%"
              
```

```c
istioctl install -f iop.yaml --skip-confirmation
```

### EKS에 Opentelemetry 설치

```c
apiVersion: v1
kind: ConfigMap
metadata:
  name: opentelemetry-collector-conf
  labels:
    app: opentelemetry-collector
data:
  opentelemetry-collector-config: |
    receivers:
      otlp:
        protocols:
          grpc:
          http: {}
    processors:
      batch: {}
      attributes:
        actions:
          - action: insert
            key: loki.attribute.labels
            value: pod, namespace, cluster, mesh
    exporters:
      loki:
        endpoint: "http://{모니터링ec2의사설ip}:3100/loki/api/v1/push" 
      logging:
        loglevel: debug
      otlp:
        endpoint: "http://{모니터링ec2의사설ip}:4317"    
        tls:
          insecure: true
      zipkin:
        endpoint: "http://{모니터링ec2의사설ip}:9411/api/v2/spans"   
        format: proto
    extensions:
      health_check: {}
      pprof: {}
      zpages: {}
    service:
      extensions: [health_check, pprof, zpages]
      pipelines:
        logs:
          receivers: [otlp]
          processors: [attributes]
          exporters: [loki, logging]
        traces:
          receivers: [otlp]
          processors: [batch]
          exporters: [otlp, zipkin]
        metrics:
          receivers: [otlp]
          processors: [batch]
          exporters: [otlp]
---
apiVersion: v1
kind: Service
metadata:
  name: opentelemetry-collector
  labels:
    app: opentelemetry-collector
spec:
  ports:
    - name: grpc-opencensus
      port: 55678
      protocol: TCP
      targetPort: 55678
    - name: grpc-otlp # Default endpoint for OpenTelemetry receiver.
      port: 4317
      protocol: TCP
      targetPort: 4317
  selector:
    app: opentelemetry-collector
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: opentelemetry-collector
spec:
  selector:
    matchLabels:
      app: opentelemetry-collector
  strategy:
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 1
    type: RollingUpdate
  template:
    metadata:
      labels:
        app: opentelemetry-collector
        sidecar.istio.io/inject: "false" # do not inject
    spec:
      containers:
        - command:
            - "/otelcol-contrib"
            - "--config=/conf/opentelemetry-collector-config.yaml"
          env:
            - name: POD_NAME
              valueFrom:
                fieldRef:
                  apiVersion: v1
                  fieldPath: metadata.name
            - name: POD_NAMESPACE
              valueFrom:
                fieldRef:
                  apiVersion: v1
                  fieldPath: metadata.namespace
          image: otel/opentelemetry-collector-contrib:0.73.0
          imagePullPolicy: IfNotPresent
          name: opentelemetry-collector
          ports:
            - containerPort: 4317
              protocol: TCP
            - name: grpc-opencensus
              containerPort: 55678
              protocol: TCP
          resources:
            limits:
              cpu: "2"
              memory: 4Gi
            requests:
              cpu: 200m
              memory: 400Mi
          terminationMessagePath: /dev/termination-log
          terminationMessagePolicy: File
          volumeMounts:
            - name: opentelemetry-collector-config-vol
              mountPath: /conf
      dnsPolicy: ClusterFirst
      restartPolicy: Always
      schedulerName: default-scheduler
      terminationGracePeriodSeconds: 30
      volumes:
        - configMap:
            defaultMode: 420
            items:
              - key: opentelemetry-collector-config
                path: opentelemetry-collector-config.yaml
            name: opentelemetry-collector-conf
          name: opentelemetry-collector-config-vol

```

### Grafana 설치

```c
# Grafana GPG 키 가져오기
wget -q -O - https://packages.grafana.com/gpg.key | sudo apt-key add -

# Grafana APT 저장소 추가
echo "deb https://packages.grafana.com/oss/deb stable main" | sudo tee /etc/apt/sources.list.d/grafana.list
sudo apt update
sudo apt install grafana
# Grafana 서비스 시작
sudo systemctl start grafana-server

# 시스템 부팅 시 Grafana 자동 시작 설정
sudo systemctl enable grafana-server
```

### Loki 설치

```c
sudo apt update
sudo apt install unzip

wget https://github.com/grafana/loki/releases/download/v2.8.1/loki-linux-amd64.zip
unzip loki-linux-amd64.zip
sudo mv loki-linux-amd64 /usr/local/bin/loki
sudo chmod +x /usr/local/bin/loki

sudo mkdir /etc/loki

sudo vi /etc/loki/local-config.yaml
===
auth_enabled: false

server:
  http_listen_port: 3100

ingester:
  lifecycler:
    address: 127.0.0.1
    ring:
      kvstore:
        store: inmemory
      replication_factor: 1
  chunk_idle_period: 5m
  chunk_retain_period: 30s
  max_transfer_retries: 0

schema_config:
  configs:
  - from: 2020-10-24
    store: boltdb-shipper
    object_store: s3
    schema: v11
    index:
      prefix: loki_index_
      period: 24h

storage_config:
  boltdb_shipper:
    active_index_directory: /tmp/loki/boltdb-shipper-active
    cache_location: /tmp/loki/boltdb-shipper-cache
    shared_store: s3
    resync_interval: 5m #이거 추가
  aws:
    s3forcepathstyle: true
    bucketnames: save-loki-log
    region: ap-northeast-2
    access_key_id: {your-aws-access-key} 
    secret_access_key: {your-aws-secret-access-key} 

limits_config:
  enforce_metric_name: false
  reject_old_samples: true
  reject_old_samples_max_age: 168h

chunk_store_config:
  max_look_back_period: 720h

table_manager:
  retention_deletes_enabled: true
  retention_period: 720h

compactor:
  working_directory: /tmp/loki/boltdb-shipper-compactor
  shared_store: s3
===

sudo mkdir -p /tmp/loki/index
sudo mkdir -p /tmp/loki/chunks
sudo chown -R ubuntu:ubuntu /tmp/loki

sudo mkdir /wal
sudo chown ubuntu:ubuntu /wal
sudo chmod 755 /wal

sudo vi /etc/systemd/system/loki.service

[Unit]
Description=Loki Log Aggregation System
After=network.target

[Service]
Type=simple
User=ubuntu
ExecStart=/usr/local/bin/loki -config.file /etc/loki/local-config.yaml
Restart=on-failure

[Install]
WantedBy=multi-user.target

sudo systemctl daemon-reload
sudo systemctl start loki
sudo systemctl status loki

# 시스템 부팅 시 Loki 자동 시작 설정
sudo systemctl enable loki

```

### Jaeger 설치

```c
sudo docker run -d --name jaeger \
  --restart=always \
  -e COLLECTOR_ZIPKIN_HOST_PORT=:9411 \
  -p 6831:6831/udp \
  -p 6832:6832/udp \
  -p 5778:5778 \
  -p 16686:16686 \
  -p 4317:4317 \
  -p 4318:4318 \
  -p 14250:14250 \
  -p 14268:14268 \
  -p 14269:14269 \
  -p 9411:9411 \
  jaegertracing/all-in-one:1.58

```

### CloudAgent 설치

```c
#1. namespace생성
kubectl apply -f https://raw.githubusercontent.com/aws-samples/amazon-cloudwatch-container-insights/latest/k8s-deployment-manifest-templates/deployment-mode/daemonset/container-insights-monitoring/cloudwatch-namespace.yaml
#2. cloud agent 서비스 계정 생성
kubectl apply -f https://raw.githubusercontent.com/aws-samples/amazon-cloudwatch-container-insights/latest/k8s-deployment-manifest-templates/deployment-mode/daemonset/container-insights-monitoring/cwagent/cwagent-serviceaccount.yaml
#3. cloud watch agent cm 생성
curl -O https://raw.githubusercontent.com/aws-samples/amazon-cloudwatch-container-insights/latest/k8s-deployment-manifest-templates/deployment-mode/daemonset/container-insights-monitoring/cwagent/cwagent-configmap-enhanced.yaml

vi cwagent-configmap-enhanced.yaml
->cluster_name을 자신의 eks name으로 수정

kubectl apply -f cwagent-configmap.yaml

#cloudwatch agent 배포
kubectl apply -f https://raw.githubusercontent.com/aws-samples/amazon-cloudwatch-container-insights/latest/k8s-deployment-manifest-templates/deployment-mode/daemonset/container-insights-monitoring/cwagent/cwagent-daemonset.yaml
#배포확인
kubectl get pods -n amazon-cloudwatch

#이때 resource request가 400으로 되어있어서 적절한 크기로 request조정
curl -O https://raw.githubusercontent.com/aws-samples/amazon-cloudwatch-container-insights/latest/k8s-deployment-manifest-templates/deployment-mode/daemonset/container-insights-monitoring/cwagent/cwagent-daemonset.yaml

```

### karpenter 노드의 메트릭을 수집 권한이 없어 아래와 같이 정책 추가후 역할 부여

![Image](https://github.com/user-attachments/assets/75f75842-cbcd-4788-80d2-8b0b90f5b6f6)

![Image](https://github.com/user-attachments/assets/df99cc39-f99c-4390-8fc3-df8b01fc27c7)

![Image](https://github.com/user-attachments/assets/4c24e223-56c7-40ce-86d8-9e7c59abc644)

![Image](https://github.com/user-attachments/assets/8a94f851-b4d8-4104-8ead-87e039f349ba)


### [Grafana dashboard 구성 - metric ]

![Image](https://github.com/user-attachments/assets/a31195e5-cc3a-447a-94f2-049fb172b72d)
