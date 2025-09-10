# Integración Prometheus → Fluentd → Elasticsearch v2

## Arquitectura de la Solución

![image.png](Integraci%C3%B3n%20Prometheus%20%E2%86%92%20Fluentd%20%E2%86%92%20Elasticsearch%20v%20263d8917ed9b8018a573c534f118e675/image.png)

---

## FASE 1: PREPARACIÓN INICIAL

### 1.1 Verificar Estado Actual

```bash
# Verificar componentes existentes
kubectl get pods -n assurance | grep -E "(prometheus|fluentd|elasticsearch)"
kubectl get svc -n assurance | grep -E "(prometheus|fluentd|elasticsearch)"

# Verificar conectividad básica
kubectl exec -n assurance $(kubectl get pods -n assurance -l app=fluentd -o jsonpath='{.items[0].metadata.name}') -- curl -s --connect-timeout 5 http://prometheus-server.assurance.svc.cluster.local:9090/api/v1/query?query=up | head -20

```

**Resultado Esperado:** Todos los servicios corriendo, Prometheus accesible.

### 1.2 Backup de Configuraciones

```bash
# Backup de configuración actual de Fluentd
kubectl get configmap fluentd-config -n assurance -o yaml > fluentd-config-backup-$(date +%Y%m%d).yaml

# Verificar backup
ls -la fluentd-config-backup-*.yaml

```

---

## FASE 2: DESPLEGAR COLECTOR DE MÉTRICAS

### 2.1 Crear ConfigMap con Script Python

```bash
kubectl apply -f - <<EOF
apiVersion: v1
kind: ConfigMap
metadata:
  name: prometheus-metrics-collector-config
  namespace: assurance
data:
  collect_and_forward.py: |
    #!/usr/bin/env python3
    import requests
    import json
    import time
    import logging
    import os
    import pytz
    from datetime import datetime
    from typing import Dict, List, Optional

    # Configuración
    PROMETHEUS_URL = "http://prometheus-server.assurance.svc.cluster.local:9090"
    FLUENTD_URL = "http://fluentd.assurance.svc.cluster.local:9888"
    COLLECTION_INTERVAL = 60
    LOG_LEVEL = os.getenv('LOG_LEVEL', 'INFO')

    # Configurar timezone de Guatemala
    guatemala_tz = pytz.timezone('America/Guatemala')

    # Setup logging con timezone de Guatemala
    logging.basicConfig(level=getattr(logging, LOG_LEVEL), format='%(asctime)s - %(levelname)s - %(message)s')
    logger = logging.getLogger(__name__)

    class PrometheusMetricsCollector:
        def __init__(self):
            self.session = requests.Session()
            self.session.timeout = 30
            
            # Definir queries de métricas (igual que antes)
            self.queries = {
                "cpu_usage_by_node": {
                    "query": '100 - (avg by (instance) (rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100)',
                    "description": "Porcentaje de uso de CPU por nodo"
                },
                "memory_usage_by_node": {
                    "query": '((node_memory_MemTotal_bytes - node_memory_MemAvailable_bytes) / node_memory_MemTotal_bytes) * 100',
                    "description": "Porcentaje de uso de memoria por nodo"
                },
                "unschedulable_pods": {
                    "query": 'kube_pod_status_scheduled{condition="false"}',
                    "description": "Pods que no pueden ser programados"
                },
                "nodes_not_ready": {
                    "query": 'kube_node_status_condition{condition="Ready",status="false"}',
                    "description": "Nodos que no están en estado Ready"
                },
                "cluster_nodes_total": {
                    "query": 'count(kube_node_info)',
                    "description": "Número total de nodos en el cluster"
                }
            }

        def query_prometheus(self, query: str) -> Optional[Dict]:
            try:
                params = {'query': query, 'time': int(time.time())}
                response = self.session.get(f"{PROMETHEUS_URL}/api/v1/query", params=params)
                response.raise_for_status()
                
                data = response.json()
                if data.get('status') == 'success':
                    return data.get('data', {})
                else:
                    logger.error(f"Prometheus query failed: {data.get('error')}")
                    return None
                    
            except Exception as e:
                logger.error(f"Error querying Prometheus: {e}")
                return None

        def format_metric(self, metric_name: str, query_info: Dict, prometheus_data: Optional[Dict]) -> Dict:
            timestamp = int(time.time())
            
            # Obtener tiempo en Guatemala
            guatemala_time = datetime.now(guatemala_tz)
            datetime_iso = guatemala_time.strftime("%Y-%m-%dT%H:%M:%S.%3fZ")
            datetime_local = guatemala_time.strftime("%Y-%m-%d %H:%M:%S %Z")
            
            base_data = {
                "timestamp": timestamp,
                "datetime": datetime_iso,
                "datetime_guatemala": datetime_local,
                "timezone": "America/Guatemala",
                "metric_name": metric_name,
                "description": query_info["description"],
                "query": query_info["query"],
                "cluster_name": "oci-oke-cluster",
                "environment": "production",
                "source": "prometheus_metrics_collector"
            }
            
            if prometheus_data and prometheus_data.get('result'):
                base_data.update({
                    "status": "success",
                    "result_count": len(prometheus_data['result']),
                    "results": prometheus_data['result']
                })
            else:
                base_data.update({
                    "status": "no_data" if prometheus_data else "error",
                    "result_count": 0,
                    "results": []
                })
                
            return base_data

        def send_to_fluentd(self, data: Dict) -> bool:
            try:
                response = self.session.post(
                    FLUENTD_URL,
                    json=data,
                    headers={'Content-Type': 'application/json'}
                )
                response.raise_for_status()
                return True
                
            except Exception as e:
                logger.error(f"Error sending to Fluentd: {e}")
                return False

        def collect_and_send_all_metrics(self) -> int:
            guatemala_time = datetime.now(guatemala_tz)
            logger.info(f"Starting metrics collection cycle - Guatemala time: {guatemala_time.strftime('%Y-%m-%d %H:%M:%S %Z')}")
            
            successful_collections = 0
            
            for metric_name, query_info in self.queries.items():
                try:
                    prometheus_data = self.query_prometheus(query_info["query"])
                    formatted_data = self.format_metric(metric_name, query_info, prometheus_data)
                    
                    if self.send_to_fluentd(formatted_data):
                        successful_collections += 1
                        logger.debug(f"Successfully collected and sent: {metric_name}")
                    else:
                        logger.warning(f"Failed to send metric: {metric_name}")
                        
                except Exception as e:
                    logger.error(f"Error processing metric {metric_name}: {e}")
                
                time.sleep(0.5)
            
            logger.info(f"Collection cycle complete: {successful_collections}/{len(self.queries)} metrics sent")
            return successful_collections

        def run_continuous(self):
            guatemala_time = datetime.now(guatemala_tz)
            logger.info(f"Starting Prometheus Metrics Collector - Guatemala time: {guatemala_time.strftime('%Y-%m-%d %H:%M:%S %Z')}")
            logger.info(f"Collection interval: {COLLECTION_INTERVAL}s")
            
            while True:
                try:
                    start_time = time.time()
                    successful_collections = self.collect_and_send_all_metrics()
                    
                    execution_time = time.time() - start_time
                    logger.info(f"Cycle completed in {execution_time:.2f}s")
                    
                    sleep_time = max(0, COLLECTION_INTERVAL - execution_time)
                    if sleep_time > 0:
                        logger.debug(f"Sleeping for {sleep_time:.2f}s")
                        time.sleep(sleep_time)
                        
                except KeyboardInterrupt:
                    logger.info("Received shutdown signal")
                    break
                except Exception as e:
                    logger.error(f"Unexpected error: {e}")
                    time.sleep(10)

    if __name__ == "__main__":
        collector = PrometheusMetricsCollector()
        
        # Test inicial con hora de Guatemala
        guatemala_time = datetime.now(guatemala_tz)
        logger.info(f"Collector starting at Guatemala time: {guatemala_time.strftime('%Y-%m-%d %H:%M:%S %Z')}")
        
        test_result = collector.query_prometheus('up')
        if test_result is None:
            logger.error("Cannot connect to Prometheus. Exiting.")
            exit(1)
        
        logger.info("Successfully connected to Prometheus")
        collector.run_continuous()
EOF
```

**Verificación:**

```bash
kubectl get configmap prometheus-metrics-collector-config -n assurance
kubectl describe configmap prometheus-metrics-collector-config -n assurance

```

### 2.2 Crear Deployment del Colector

```bash
kubectl apply -f - <<EOF
apiVersion: apps/v1
kind: Deployment
metadata:
  name: prometheus-metrics-collector
  namespace: assurance
  labels:
    app: prometheus-metrics-collector
spec:
  replicas: 1
  selector:
    matchLabels:
      app: prometheus-metrics-collector
  template:
    metadata:
      labels:
        app: prometheus-metrics-collector
    spec:
      containers:
      - name: collector
        image: python:3.9-slim
        command: ["/bin/sh"]
        args:
          - -c
          - |
            pip install requests --quiet
            pip install pytz --quiet
            python3 /app/collect_and_forward.py
        env:
        - name: PROMETHEUS_URL
          value: "http://prometheus-server.assurance.svc.cluster.local:9090"
        - name: FLUENTD_URL
          value: "http://fluentd.assurance.svc.cluster.local:9888"
        - name: LOG_LEVEL
          value: "INFO"
        - name: PYTHONUNBUFFERED
          value: "1"
        volumeMounts:
        - name: config
          mountPath: /app
        resources:
          requests:
            memory: "64Mi"
            cpu: "50m"
          limits:
            memory: "128Mi"
            cpu: "100m"
        livenessProbe:
          exec:
            command:
            - python3
            - -c
            - "import requests; requests.get('http://prometheus-server.assurance.svc.cluster.local:909/api/v1/query?query=up', timeout=10)"
          initialDelaySeconds: 60
          periodSeconds: 120
        readinessProbe:
          exec:
            command:
            - python3
            - -c
            - "import requests; requests.get('http://prometheus-server.assurance.svc.cluster.local:9090/api/v1/query?query=up', timeout=5)"
          initialDelaySeconds: 30
          periodSeconds: 60
      volumes:
      - name: config
        configMap:
          name: prometheus-metrics-collector-config
      restartPolicy: Always
EOF

```

**Verificación:**

```bash
kubectl get deployment prometheus-metrics-collector -n assurance
kubectl get pods -n assurance -l app=prometheus-metrics-collector
kubectl logs -n assurance -l app=prometheus-metrics-collector --tail=20

```

---

## FASE 3: CONFIGURAR FLUENTD PARA RECIBIR HTTP

### 3.1 Crear Servicio para Fluentd

```bash
kubectl apply -f - <<EOF
apiVersion: v1
kind: Service
metadata:
  name: fluentd
  namespace: assurance
  labels:
    app: fluentd
spec:
  selector:
    app: fluentd
  ports:
  - port: 9888
    targetPort: 9888
    name: http-input
    protocol: TCP
  - port: 24231
    targetPort: 24231
    name: monitor
    protocol: TCP
  type: ClusterIP
EOF

```

### 3.2 Actualizar ConfigMap de Fluentd

```bash
# Obtener configuración actual
kubectl get configmap fluentd-config -n assurance -o yaml > current-fluentd-config.yaml

# Crear nueva configuración con HTTP input
kubectl apply -f - <<EOF
apiVersion: v1
kind: ConfigMap
metadata:
  name: fluentd-config
  namespace: assurance
data:
  fluent.conf: |
    <source>
      @type monitor_agent
      bind 0.0.0.0
      port 24231
    </source>

    <label @FLUENT_LOG>
      <match fluent.**>
        @type null
      </match>
    </label>

    # === HTTP INPUT PARA COLECTOR DE MÉTRICAS ===
    <source>
      @type http
      @id prometheus_metrics_input
      tag prometheus.collected
      port 9888
      bind 0.0.0.0
      body_size_limit 32m
      keepalive_timeout 10s
      add_http_headers false
      <parse>
        @type json
        time_key timestamp
        time_type unixtime
      </parse>
    </source>

    # Filtro para enriquecer métricas del colector
    <filter prometheus.collected>
      @type record_transformer
      enable_ruby true
      <record>
        received_at \${Time.now.strftime("%Y-%m-%dT%H:%M:%S.%3NZ")}
        fluentd_node "\#{ENV['K8S_NODE_NAME']}"
        source_type "external_collector"
      </record>
    </filter>

    # Output para métricas del colector a Elasticsearch
    <match prometheus.collected>
      @type elasticsearch
      @id out_es_prometheus_collected
      @log_level info
      include_tag_key true
      host "\#{ENV['FLUENT_ELASTICSEARCH_HOST'] || 'elasticsearch-prod-es-internal-http.assurance.svc.cluster.local'}"
      port "\#{ENV['FLUENT_ELASTICSEARCH_PORT'] || '9200'}"
      scheme "\#{ENV['FLUENT_ELASTICSEARCH_SCHEME'] || 'https'}"
      ssl_verify "\#{ENV['FLUENT_ELASTICSEARCH_SSL_VERIFY'] || 'false'}"
      ssl_version "\#{ENV['FLUENT_ELASTICSEARCH_SSL_VERSION'] || 'TLSv1_2'}"
      user "\#{ENV['FLUENT_ELASTICSEARCH_USER'] || use_default}"
      password "\#{ENV['FLUENT_ELASTICSEARCH_PASSWORD'] || use_default}"

      logstash_prefix "prometheus-collected-metrics"
      logstash_dateformat "%Y.%m.%d"
      logstash_format true
      index_name "prometheus-collected-metrics-%Y.%m.%d"
      type_name "collected_prometheus_metric"

      <buffer>
        @type file
        path /var/log/fluentd-buffers/prometheus.collected.buffer
        flush_thread_count 2
        flush_interval 20s
        chunk_limit_size 2M
        queue_limit_length 32
        retry_max_interval 30
        retry_forever true
      </buffer>
    </match>

    # === CONFIGURACIÓN EXISTENTE DE LOGS ===
    <match kubernetes.var.log.containers.**fluentd**.log>
      @type null
    </match>

    <match kubernetes.var.log.containers.**kube-system**.log>
      @type null
    </match>

    <match kubernetes.var.log.containers.**kibana**.log>
      @type null
    </match>

    <source>
      @type tail
      path /var/log/containers/*.log
      pos_file fluentd-docker.pos
      tag kubernetes.*
      <parse>
        @type multi_format
        <pattern>
          format json
          time_key time
          time_type string
          time_format "%Y-%m-%dT%H:%M:%S.%NZ"
          keep_time_key false
        </pattern>
        <pattern>
          format regexp
          expression /^(?\<time\>[^ ]+) (?\<stream\>stdout|stderr)( (?\<logtag\>[^ ]+))? (?\<log\>.*)$/
          time_format '%Y-%m-%dT%H:%M:%S.%N%:z'
          keep_time_key false
        </pattern>
        <pattern>
          format none
          message_key log
        </pattern>
      </parse>
    </source>

    <filter kubernetes.var.log.containers.**>
      @type kubernetes_metadata
      @id filter_kube_metadata
    </filter>

    <filter kubernetes.var.log.containers.**>
      @type parser
      <parse>
        @type json
        time_key time
        time_type string
        time_format "%Y-%m-%dT%H:%M:%S.%NZ"
        keep_time_key false
      </parse>
      key_name log
      replace_invalid_sequence true
      emit_invalid_record_to_error true
      reserve_data true
    </filter>

    <filter kubernetes.var.log.containers.**>
      @type grep
      <exclude>
        key \$.kubernetes.namespace_name
        pattern /^(argo-rollouts|argocd|assurance|commvault|data-services|elastic-system|falco|hpa-test|infinispan|iptables-istio|istio-ingress|istio-system|kafka|kube-node-lease|kube-public|kube-system|lifecycle|nifi|nifi-cluster|observability|policy|rabbitmq|redis|scaling|security|trivy|vault)$/
      </exclude>
    </filter>

    <match **>
        @type elasticsearch
        @id out_es
        @log_level info
        include_tag_key true
        host "\#{ENV['FLUENT_ELASTICSEARCH_HOST'] || 'elasticsearch-prod-es-internal-http.assurance.svc.cluster.local'}"
        port "\#{ENV['FLUENT_ELASTICSEARCH_PORT'] || '9200'}"
        path "\#{ENV['FLUENT_ELASTICSEARCH_PATH']}"
        scheme "\#{ENV['FLUENT_ELASTICSEARCH_SCHEME'] || 'https'}"
        ssl_verify "\#{ENV['FLUENT_ELASTICSEARCH_SSL_VERIFY'] || 'false'}"
        ssl_version "\#{ENV['FLUENT_ELASTICSEARCH_SSL_VERSION'] || 'TLSv1_2'}"
        user "\#{ENV['FLUENT_ELASTICSEARCH_USER'] || use_default}"
        password "\#{ENV['FLUENT_ELASTICSEARCH_PASSWORD'] || use_default}"
        reload_connections "\#{ENV['FLUENT_ELASTICSEARCH_RELOAD_CONNECTIONS'] || 'false'}"
        reconnect_on_error "\#{ENV['FLUENT_ELASTICSEARCH_RECONNECT_ON_ERROR'] || 'true'}"
        reload_on_failure "\#{ENV['FLUENT_ELASTICSEARCH_RELOAD_ON_FAILURE'] || 'true'}"
        log_es_400_reason "\#{ENV['FLUENT_ELASTICSEARCH_LOG_ES_400_REASON'] || 'false'}"
        logstash_prefix "\#{ENV['FLUENT_ELASTICSEARCH_LOGSTASH_PREFIX'] || 'kubernetes-logs-prod'}"
        logstash_dateformat "\#{ENV['FLUENT_ELASTICSEARCH_LOGSTASH_DATEFORMAT'] || '%Y.%m.%d'}"
        logstash_format "\#{ENV['FLUENT_ELASTICSEARCH_LOGSTASH_FORMAT'] || 'true'}"
        index_name "\#{ENV['FLUENT_ELASTICSEARCH_LOGSTASH_INDEX_NAME'] || 'kubernetes-logs-prod'}"
        type_name "\#{ENV['FLUENT_ELASTICSEARCH_LOGSTASH_TYPE_NAME'] || 'fluentd'}"
        include_timestamp "\#{ENV['FLUENT_ELASTICSEARCH_INCLUDE_TIMESTAMP'] || 'false'}"
        template_name "\#{ENV['FLUENT_ELASTICSEARCH_TEMPLATE_NAME'] || use_nil}"
        template_file "\#{ENV['FLUENT_ELASTICSEARCH_TEMPLATE_FILE'] || use_nil}"
        template_overwrite "\#{ENV['FLUENT_ELASTICSEARCH_TEMPLATE_OVERWRITE'] || use_default}"
        sniffer_class_name "\#{ENV['FLUENT_SNIFFER_CLASS_NAME'] || 'Fluent::Plugin::ElasticsearchSimpleSniffer'}"
        request_timeout "\#{ENV['FLUENT_ELASTICSEARCH_REQUEST_TIMEOUT'] || '30s'}"
        <buffer>
          @type file
          path /var/log/fluentd-buffers/kubernetes.system.buffer
          flush_thread_count "\#{ENV['FLUENT_ELASTICSEARCH_BUFFER_FLUSH_THREAD_COUNT'] || '8'}"
          flush_interval "\#{ENV['FLUENT_ELASTICSEARCH_BUFFER_FLUSH_INTERVAL'] || '10s'}"
          chunk_limit_size "\#{ENV['FLUENT_ELASTICSEARCH_BUFFER_CHUNK_LIMIT_SIZE'] || '2M'}"
          queue_limit_length "\#{ENV['FLUENT_ELASTICSEARCH_BUFFER_QUEUE_LIMIT_LENGTH'] || '32'}"
          retry_max_interval "\#{ENV['FLUENT_ELASTICSEARCH_BUFFER_RETRY_MAX_INTERVAL'] || '30'}"
          retry_forever true
        </buffer>
      </match>
EOF

```

### 3.3 Actualizar DaemonSet de Fluentd

```bash
# Agregar puerto 9888 al DaemonSet
kubectl patch daemonset fluentd -n assurance --patch '
spec:
  template:
    spec:
      containers:
      - name: fluentd
        ports:
        - containerPort: 9888
          name: http-input
          protocol: TCP
        - containerPort: 24231
          name: monitor
          protocol: TCP
'

# Reiniciar DaemonSet para aplicar cambios
kubectl rollout restart daemonset/fluentd -n assurance

```

**Verificación:**

```bash
kubectl rollout status daemonset/fluentd -n assurance
kubectl get pods -n assurance -l app=fluentd
kubectl describe daemonset fluentd -n assurance | grep -A10 "Ports:"

```

---

## FASE 4: VERIFICACIÓN Y TESTING

### 4.1 Verificar Conectividad entre Componentes

```bash
# Estado del colector
kubectl get pods -n assurance -l app=prometheus-metrics-collector
kubectl logs -n assurance -l app=prometheus-metrics-collector --tail=50

# Estado de Fluentd
kubectl get pods -n assurance -l app=fluentd
kubectl logs -n assurance -l app=fluentd --tail=50 | grep -E "(prometheus|9888)"

# Servicios
kubectl get svc -n assurance | grep -E "(fluentd|prometheus)"

```

### 4.2 Test Manual de Conectividad

```bash
# Test desde el colector hacia Prometheus
kubectl exec -n assurance deployment/prometheus-metrics-collector -- python3 -c "
import requests
try:
    r = requests.get('http://prometheus-server.assurance.svc.cluster.local:9090/api/v1/query?query=up', timeout=10)
    print(f'Prometheus status: {r.status_code}')
    print(f'Response preview: {r.text[:200]}')
except Exception as e:
    print(f'Error: {e}')
"

# Test desde el colector hacia Fluentd
kubectl exec -n assurance deployment/prometheus-metrics-collector -- python3 -c "
import requests, json
try:
    test_data = {'test': 'connectivity', 'timestamp': 1234567890}
    r = requests.post('http://fluentd.assurance.svc.cluster.local:9888',
                     json=test_data, timeout=10)
    print(f'Fluentd status: {r.status_code}')
except Exception as e:
    print(f'Error: {e}')
"

```

### 4.3 Verificar en Elasticsearch

```bash
# Obtener credenciales de Elasticsearch
ELASTIC_PASSWORD=$(kubectl get secret -n assurance elasticsearch-prod-es-elastic-user -o jsonpath='{.data.elastic}' | base64 -d 2>/dev/null || echo "your-password")

# Verificar índices creados
kubectl exec -n assurance elasticsearch-prod-es-data-0 -- curl -k -u elastic:${ELASTIC_PASSWORD} "https://localhost:9200/_cat/indices?v" | grep collected

# Ver documentos recientes
kubectl exec -n assurance elasticsearch-prod-es-data-0 -- curl -k -u elastic:${ELASTIC_PASSWORD} "https://localhost:9200/prometheus-collected-metrics-*/_search?size=3&sort=timestamp:desc&pretty"

# Contar documentos por métrica
kubectl exec -n assurance elasticsearch-prod-es-data-0 -- curl -k -u elastic:${ELASTIC_PASSWORD} "https://localhost:9200/prometheus-collected-metrics-*/_search?size=0&aggs={\"by_metric\":{\"terms\":{\"field\":\"metric_name.keyword\"}}}&pretty"

```

**Resultado Esperado:** Índices `prometheus-collected-metrics-YYYY.MM.DD`, documentos con métricas estructuradas.

---

## FASE 5: CONFIGURACIÓN EN KIBANA/GRAFANA

### 5.1 Index Pattern en Kibana

1. Acceder a Kibana: `http://kibana-url`
2. Management → Stack Management → Index Patterns
3. Crear pattern: `prometheus-collected-metrics-*`
4. Time field: `timestamp` o `datetime`
5. Guardar index pattern

### 6.2 Queries de Ejemplo para Kibana

```json
# CPU usage por nodo (últimas 2 horas)
{
  "query": {
    "bool": {
      "must": [
        {"term": {"metric_name": "cpu_usage_by_node"}},
        {"term": {"status": "success"}},
        {"range": {"timestamp": {"gte": "now-2h"}}}
      ]
    }
  },
  "aggs": {
    "by_node": {
      "terms": {
        "field": "results.metric.instance.keyword",
        "size": 10
      },
      "aggs": {
        "latest_value": {
          "top_hits": {
            "sort": [{"timestamp": {"order": "desc"}}],
            "size": 1,
            "_source": ["results.value", "timestamp"]
          }
        }
      }
    }
  }
}

# Pods no programables
{
  "query": {
    "bool": {
      "must": [
        {"term": {"metric_name": "unschedulable_pods"}},
        {"range": {"timestamp": {"gte": "now-1h"}}}
      ]
    }
  },
  "sort": [{"timestamp": {"order": "desc"}}]
}

```

---

## MÉTRICAS DISPONIBLES

| Categoría | Métrica | Descripción |
| --- | --- | --- |
| **CPU** | `cpu_usage_by_node` | % uso CPU por nodo |
|  | `cpu_allocatable_by_node` | CPU asignable por nodo |
|  | `cpu_requests_by_node` | CPU solicitado por nodo |
| **Memoria** | `memory_usage_by_node` | % uso memoria por nodo |
|  | `memory_total_by_node` | Memoria total por nodo |
|  | `memory_allocatable_by_node` | Memoria asignable por nodo |
|  | `memory_requests_by_node` | Memoria solicitada por nodo |
| **Pods** | `unschedulable_pods` | Pods no programables |
|  | `pending_pods` | Pods pendientes |
|  | `pending_pods_by_namespace` | Pods pendientes por namespace |
| **Nodos** | `nodes_not_ready` | Nodos no listos |
|  | `nodes_memory_pressure` | Nodos con presión memoria |
|  | `nodes_disk_pressure` | Nodos con presión disco |
| **Cluster** | `cluster_nodes_total` | Total nodos |
|  | `cluster_pods_total` | Total pods |
|  | `cluster_cpu_allocatable_total` | Total CPU disponible |

---