# VMware Container Storage Interface (CSI) Addon для Talos Kubernetes

- [VMware Container Storage Interface (CSI) Addon для Talos Kubernetes](#vmware-container-storage-interface-csi-addon-для-talos-kubernetes)
  - [Зачем нужен этот аддон](#зачем-нужен-этот-аддон)
  - [Требования](#требования)
  - [Конфигурация](#конфигурация)
    - [Ключевые переменные (в `ansible/<cluster-name>.yml`)](#ключевые-переменные-в-ansiblecluster-nameyml)
  - [Особенности реализации](#особенности-реализации)
  - [Валидация установки](#валидация-установки)
  - [Пример использования в приложении](#пример-использования-в-приложении)
  - [Производственные рекомендации](#производственные-рекомендации)

## Зачем нужен этот аддон

VMware CSI обеспечивает **нативную интеграцию Kubernetes с системами хранения vSphere**, предоставляя кластеру возможность:

- Динамического создания Persistent Volumes (PV) из vSphere datastore
- Автоматического связывания PersistentVolumeClaims (PVC) с реальными томами
- Поддержки различных политик хранения (Storage Policies)
- Интеграции с vSAN для enterprise-grade отказоустойчивости
- Клонирования томов и создания снапшотов
- Работы с топологией (привязка томов к конкретным зонам/хостам)

**Без этого аддона:**  
Stateful приложения не смогут хранить данные на внешних томах, а динамическое выделение хранилища будет невозможно.

## Требования

| Компонент      | Версия/Требование                  |
| -------------- | ---------------------------------- |
| VMware vSphere | 7.0 U3+                            |
| vCenter Server | 7.0 U3d+                           |
| Talos Cluster  | С установленным vmware-cpi аддоном |
| Helm           | >= 3.8                             |

## Конфигурация

### Ключевые переменные (в `ansible/<cluster-name>.yml`)

```yaml
# vCenter подключение
vcenter_hostname: "vcenter-01.infra.corp"
vcenter_cpi_user: "svc-k8s-vcenter@okbtsp.local"
vcenter_cpi_password: !vault | ...  # Обязательно в Vault!
vcenter_datacenters:
  - "Datacenter"

# Helm настройки
vsphere_csi_helm_repo_url: "https://vsphere-tmm.github.io/helm-charts"
vsphere_csi_chart_ref: "vsphere-tmm/vsphere-csi"
vsphere_csi_chart_version: "3.8.1"
vsphere_csi_namespace: "vmware-csi"

# StorageClass конфигурация
vsphere_csi_storageclass_enabled: true
vsphere_csi_storageclass_default: true
vsphere_csi_storage_policy_name: "VM Storage Policy for vSAN-Prod"
vsphere_csi_reclaim_policy: "Retain"  # Варианты: Delete, Retain
vsphere_csi_volume_binding_mode: "WaitForFirstConsumer"  # или Immediate
vsphere_csi_datastore_url: "ds:///vmfs/volumes/vsan:52b8e767c5d62dc2-19513fa231fbcea1/"
```

## Особенности реализации

- **Безопасность:**  
  Namespace `vmware-csi` помечен как `privileged` для PodSecurityPolicy, что необходимо для работы CSI драйвера.

- **Топология:**  
  Используется режим `WaitForFirstConsumer`, при котором том создается на том же хосте, где запущен под, что оптимизирует производительность и соответствие политикам.

- **Отказоустойчивость:**

  - Controller развертывается как Deployment с несколькими репликами
  - Node plugin развертывается как DaemonSet на всех нодах
  - Долгие таймауты ожидания (до 30 минут) для надежности в больших кластерах

- **Интеграция с vSAN:**  
  Используется Storage Policy для vSAN, что обеспечивает enterprise-grade отказоустойчивость и производительность.

## Валидация установки

```bash
# Проверка состояния CSI компонентов
kubectl get pods -n vmware-csi
kubectl get storageclasses
kubectl get csinodes

# Тестовое создание PVC
kubectl apply -f - <<EOF
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: test-pvc
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
EOF

# Проверка привязки PVC
kubectl get pvc test-pvc
```

## Пример использования в приложении

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: database
spec:
  serviceName: "database"
  replicas: 3
  template:
    spec:
      containers:
        - name: db
          image: postgres:15
          volumeMounts:
            - name: data
              mountPath: /var/lib/postgresql/data
  volumeClaimTemplates:
    - metadata:
        name: data
      spec:
        accessModes: ["ReadWriteOnce"]
        storageClassName: "vsphere-csi"
        resources:
          requests:
            storage: 10Gi
```

## Производственные рекомендации

1. **Storage Policies:**  
   Создайте отдельные StorageClass для разных SLA (performance, capacity, durability).

2. **Reclaim Policy:**  
   Используйте `Retain` для production-баз данных и критичных данных для предотвращения случайного удаления.

3. **Топология:**  
   Для multi-AZ кластеров настройте топологические ключи в StorageClass:

   ```yaml
   volumeBindingMode: WaitForFirstConsumer
   allowedTopologies:
     - matchLabelExpressions:
         - key: topology.kubernetes.io/zone
           values: [zone-a, zone-b]
   ```

4. **Мониторинг:**  
   Настройте сбор метрик с CSI контроллера для отслеживания операций создания/удаления томов.

> **Важно:** CSI драйвер **требует предварительной установки** VMware CPI (vmware-cpi) для корректной работы с топологией и метаданными vSphere. Никогда не используйте одинаковые сервисные аккаунты для CPI и CSI в production-средах.
