## **Como implementar Lokistack no OpenShift projetado para suportar ambientes de grande porte**

Este guia apresenta um passo a passo para implantar o LokiStack no OpenShift em ambientes de grande porte (mais de 10.000 namespaces e 300 nodes), incluindo instruções de migração do stack EFK (Elasticsearch, Fluentd, Kibana) para o Loki.

O LokiStack se tornou a solução recomendada para observabilidade de logs no OpenShift, substituindo o tradicional stack **EFK (Elasticsearch, Fluentd, Kibana)**.  
Sua arquitetura escalável e o consumo otimizado de recursos tornam o Loki ideal para **ambientes de grande porte**, com milhares de namespaces e centenas de nodes.

### Ambiente de referência

* **OpenShift:** 4.16
* **Instalação:** vSphere IPI
* **Lokistack:** versão 6.2

---

## 1. Migração do EFK para Loki

Se o seu ambiente utiliza o *OpenShift Logging* baseado em Elasticsearch e você deseja migrar para o Loki, siga os passos abaixo.

### 1.1. Tornar o Cluster Logging não gerenciado

Isso evita que o operador sobrescreva configurações durante a migração.

```bash
oc -n openshift-logging patch clusterlogging/instance -p '{"spec":{"managementState": "Unmanaged"}}' --type=merge
oc -n openshift-logging patch elasticsearch/elasticsearch -p '{"metadata":{"ownerReferences": []}}' --type=merge
oc -n openshift-logging patch kibana/kibana -p '{"metadata":{"ownerReferences": []}}' --type=merge
```

### 1.2. Remover subscriptions e operators antigos

Elimine as *Subscriptions* e os *ClusterServiceVersions* (CSVs) associados ao operador **Cluster Logging**.

```bash
oc -n openshift-logging delete subs cluster-logging 
oc -n openshift-logging delete csv -l operators.coreos.com/cluster-logging.openshift-logging
```

#### Reciclar os pods dos namespaces `openshift-marketplace` e `openshift-operator-lifecycle-manager`

```bash
oc -n openshift-marketplace delete pod --all
oc -n openshift-operator-lifecycle-manager delete pod --all
```

> **Nota:**
> Reciclar os pods de `openshift-marketplace` e `openshift-operator-lifecycle-manager` evita erros de cache e falhas em instalações subsequentes de operators.

### 1.3. Fazer backup dos CRs do Elasticsearch e Kibana

Recomendado caso deseje manter os dados por alguns dias antes da desativação.

```bash
oc -n openshift-logging get elasticsearch elasticsearch -o yaml > cr-elasticsearch.yaml
oc -n openshift-logging get kibana kibana -o yaml > cr-kibana.yaml
```

### 1.4. Remover os *collectors* do Elasticsearch

```bash
oc -n openshift-logging delete ds collector
```

---

## 2. (Opcional) Criar nodes dedicados para o Loki

Para ambientes grandes, é altamente recomendado criar **MachineSets** e **MachineConfigPools** específicos para os componentes do Loki.
Isso garante isolamento de carga e previsibilidade de desempenho.

Ajuste os campos conforme o seu ambiente.

```yaml
# MachineSet para nós dedicados ao Loki
apiVersion: machine.openshift.io/v1beta1
kind: MachineSet
metadata:
  labels:
    machine.openshift.io/cluster-api-cluster: <infrastructure_id>
  name: <infrastructure_id>-<role>
  namespace: openshift-machine-api
spec:
  replicas: 3
  selector:
    matchLabels:
      machine.openshift.io/cluster-api-cluster: <infrastructure_id>
  template:
    metadata:
      labels:
        machine.openshift.io/cluster-api-cluster: <infrastructure_id>
        machine.openshift.io/cluster-api-machine-role: <role>
        machine.openshift.io/cluster-api-machine-type: <role>
        machine.openshift.io/cluster-api-machineset: <infrastructure_id>-<role>
    spec:
      metadata:
        labels:
          node-role.kubernetes.io/logging: ""
      providerSpec:
        value:
          apiVersion: machine.openshift.io/v1beta1
          kind: VSphereMachineProviderSpec
          credentialsSecret:
            name: vsphere-cloud-credentials
          diskGiB: 120
          memoryMiB: 65536
          numCPUs: 24
          numCoresPerSocket: 2
          template: <vm_template_name>
          userDataSecret:
            name: worker-user-data
          network:
            devices:
            - networkName: "<vm_network_name>"
          workspace:
            datacenter: <vcenter_data_center_name>
            datastore: <vcenter_datastore_name>
            folder: <vcenter_vm_folder_path>
            resourcepool: <vsphere_resource_pool>
            server: <vcenter_server_ip>
      taints:
      - effect: NoSchedule
        key: node-role.kubernetes.io/infra
      - effect: NoSchedule
        key: node-role.kubernetes.io/logging
---
# MachineConfigPool correspondente
apiVersion: machineconfiguration.openshift.io/v1
kind: MachineConfigPool
metadata:
  name: logging
spec:
  machineConfigSelector:
    matchExpressions:
      - {key: machineconfiguration.openshift.io/role, operator: In, values: [worker,logging]}
  nodeSelector:
    matchLabels:
      node-role.kubernetes.io/logging: ""
```

> **Dica:**
> Isolar os componentes do Loki evita competição de recursos com workloads de aplicações.

---

## 3. Implantar o LokiStack e componentes necessários

Antes de aplicar o manifest do LokiStack, é necessário criar o **bucket** que servirá como backend de armazenamento e disponibilizar suas credenciais e certificados.

### Criando o bucket

Crie o bucket no provedor de armazenamento desejado (ex.: **AWS S3**, **MinIO**, **Ceph RGW**, etc.).
Anote:

* Nome do bucket
* Endpoint
* Access Key e Secret Key
* Região
* CA (caso use certificado customizado)

### Criando o ConfigMap com a CA do bucket

O arquivo `ca-bucket-loki.crt` deve conter a CA utilizada pelo endpoint do bucket.

```bash
oc create configmap custom-ca-configmap \
  --from-file=service-ca.crt=ca-bucket-loki.crt \
  -n openshift-logging
```

> **Explicação:**
> Esse ConfigMap será referenciado pelo LokiStack para estabelecer uma conexão TLS confiável com o bucket.

### Criando a Secret com as credenciais do bucket

Ajuste os valores conforme o seu bucket:

```bash
oc create secret generic lokistack-s3 \
  --from-literal=bucketnames="<bucket_name>" \
  --from-literal=endpoint="<aws_bucket_endpoint>" \
  --from-literal=access_key_id="<aws_access_key_id>" \
  --from-literal=access_key_secret="<aws_access_key_secret>" \
  --from-literal=region="<aws_region>" \
  -n openshift-logging
```

> **Dica:**
> Verifique se as credenciais possuem permissão de escrita e leitura no bucket configurado.

---

### Aplicando os manifests

Após criar o bucket, o ConfigMap e a Secret, aplique os YAMLs a seguir.
Eles devem ser aplicados em ordem, conforme a *sync-wave* definida (boa prática para uso com ArgoCD).

#### 3.1. Namespaces

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: openshift-operators-redhat
  annotations:
    argocd.argoproj.io/sync-wave: "0"
  labels:
    openshift.io/cluster-monitoring: "true"
---
apiVersion: v1
kind: Namespace
metadata:
  name: openshift-logging
  annotations:
    argocd.argoproj.io/sync-wave: "0"
  labels:
    openshift.io/cluster-monitoring: "true"
---
apiVersion: v1
kind: Namespace
metadata:
  name: openshift-cluster-observability-operator
  annotations:
    argocd.argoproj.io/sync-wave: "0"
  labels:
    openshift.io/cluster-monitoring: "true"
```

---

### 3.2. OperatorGroups

```yaml
apiVersion: operators.coreos.com/v1
kind: OperatorGroup
metadata:
  name: openshift-operators-redhat
  namespace: openshift-operators-redhat
  annotations:
    argocd.argoproj.io/sync-wave: "1"
spec:
  upgradeStrategy: Default
---
apiVersion: operators.coreos.com/v1
kind: OperatorGroup
metadata:
  name: cluster-logging
  namespace: openshift-logging
  annotations:
    argocd.argoproj.io/sync-wave: "1"
spec:
  upgradeStrategy: Default
---
apiVersion: operators.coreos.com/v1
kind: OperatorGroup
metadata:
  name: cluster-observability-operator
  namespace: openshift-cluster-observability-operator
  annotations:
    argocd.argoproj.io/sync-wave: "1"
spec:
  upgradeStrategy: Default
```

---

### 3.3. Subscriptions

As *Subscriptions* instalam os operadores Loki, Cluster Logging e Cluster Observability Operator a partir do catálogo oficial.

```yaml
apiVersion: operators.coreos.com/v1alpha1
kind: Subscription
metadata:
  name: loki-operator
  namespace: openshift-operators-redhat
  annotations:
    argocd.argoproj.io/sync-wave: "2"
spec:
  channel: stable-6.2
  installPlanApproval: Automatic
  name: loki-operator
  source: redhat-operators 
  sourceNamespace: openshift-marketplace
---
apiVersion: operators.coreos.com/v1alpha1
kind: Subscription
metadata:
  name: cluster-logging
  namespace: openshift-logging
  annotations:
    argocd.argoproj.io/sync-wave: "2"
spec:
  channel: stable-6.2
  installPlanApproval: Automatic
  name: cluster-logging
  source: redhat-operators 
  sourceNamespace: openshift-marketplace
---
apiVersion: operators.coreos.com/v1alpha1
kind: Subscription
metadata:
  name: cluster-observability-operator
  namespace: openshift-cluster-observability-operator
  annotations:
    argocd.argoproj.io/sync-wave: "2"
spec:
  channel: stable
  installPlanApproval: Automatic
  name: cluster-observability-operator
  source: redhat-operators
  sourceNamespace: openshift-marketplace
```

> **Boas práticas:**
>
> * Utilize `installPlanApproval: Automatic` apenas em ambientes de teste.
> * Em produção, prefira `Manual` para maior controle de upgrades.

---

### 3.4. ServiceAccount e permissões RBAC

Crie a ServiceAccount utilizada pelo *collector* e conceda as permissões necessárias para escrita e coleta de logs de aplicações, infraestrutura e auditoria.

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  annotations:
    argocd.argoproj.io/sync-wave: "2"
  name: logging-collector
  namespace: openshift-logging
```

ClusterRoleBindings associados:

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  annotations:
    argocd.argoproj.io/sync-wave: "3"
  name: logging-collector-logs-writer
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: logging-collector-logs-writer
subjects:
- kind: ServiceAccount
  name: logging-collector
  namespace: openshift-logging
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  annotations:
    argocd.argoproj.io/sync-wave: "3"      
  name: collect-application-logs
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: collect-application-logs
subjects:
- kind: ServiceAccount
  name: logging-collector
  namespace: openshift-logging
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  annotations:
    argocd.argoproj.io/sync-wave: "3"      
  name: collect-infrastructure-logs
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: collect-infrastructure-logs
subjects:
- kind: ServiceAccount
  name: logging-collector
  namespace: openshift-logging
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  annotations:
    argocd.argoproj.io/sync-wave: "3"      
  name: collect-audit-logs
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: collect-audit-logs
subjects:
- kind: ServiceAccount
  name: logging-collector
  namespace: openshift-logging
```

---

### 3.5. LokiStack CR

Defina a instância do LokiStack com configuração otimizada para ambientes grandes:

```yaml
apiVersion: loki.grafana.com/v1
kind: LokiStack    
metadata:
  annotations:
    argocd.argoproj.io/sync-wave: "4"
    argocd.argoproj.io/sync-options: SkipDryRunOnMissingResource=true
  name: lokistack
  namespace: openshift-logging
spec:
  hashRing:
    memberlist:
      instanceAddrType: podIP
    type: memberlist
  limits:
    global:
      ingestion:
        maxLabelNameLength: 1024
        perStreamRateLimitBurst: 1048576
        ingestionRate: 1048576
        maxLabelValueLength: 2048
        maxLineSize: 512000
        maxGlobalStreamsPerTenant: 150000
        maxLabelNamesPerSeries: 32
        ingestionBurstSize: 2097152
        perStreamRateLimit: 524288
      queries:
        maxChunksPerQuery: 10000
        maxEntriesLimitPerQuery: 50000
        maxQuerySeries: 5000
        queryTimeout: 1m30s
      retention:
        days: 7
        streams:
          - days: 7
            priority: 1
            selector: '{log_type="application"}'
          - days: 7
            priority: 1
            selector: '{log_type="infrastructure"}'
          - days: 7
            priority: 1
            selector: '{log_type="audit"}'
  managementState: Managed
  replicationFactor: 2
  rules:
    enabled: false
  size: 1x.medium
  storage:
    schemas:
    - effectiveDate: "2025-08-08"
      version: v13
    secret:
      name: lokistack-s3
      type: s3
    tls:
      caName: custom-ca-configmap
  storageClassName: vsphere-csi
  template:
    compactor:
      nodeSelector:
        node-role.kubernetes.io/logging
      tolerations:
      - effect: NoSchedule
        key: node-role.kubernetes.io/infra
      - effect: NoSchedule
        key: node-role.kubernetes.io/logging
    distributor:
      nodeSelector:
        node-role.kubernetes.io/logging
      tolerations:
      - effect: NoSchedule
        key: node-role.kubernetes.io/infra
      - effect: NoSchedule
        key: node-role.kubernetes.io/logging
    gateway:
      nodeSelector:
        node-role.kubernetes.io/logging
      replicas: 3
      tolerations:
      - effect: NoSchedule
        key: node-role.kubernetes.io/infra
      - effect: NoSchedule
        key: node-role.kubernetes.io/logging
    indexGateway:
      nodeSelector:
        node-role.kubernetes.io/logging
      tolerations:
      - effect: NoSchedule
        key: node-role.kubernetes.io/infra
      - effect: NoSchedule
        key: node-role.kubernetes.io/logging
    ingester:
      nodeSelector:
        node-role.kubernetes.io/logging
      replicas: 3
      tolerations:
      - effect: NoSchedule
        key: node-role.kubernetes.io/infra
      - effect: NoSchedule
        key: node-role.kubernetes.io/logging
    querier:
      nodeSelector:
        node-role.kubernetes.io/logging
      tolerations:
      - effect: NoSchedule
        key: node-role.kubernetes.io/infra
      - effect: NoSchedule
        key: node-role.kubernetes.io/logging
    queryFrontend:
      nodeSelector:
        node-role.kubernetes.io/logging
      tolerations:
      - effect: NoSchedule
        key: node-role.kubernetes.io/infra
      - effect: NoSchedule
        key: node-role.kubernetes.io/logging
    ruler:
      nodeSelector:
        node-role.kubernetes.io/logging
      tolerations:
      - effect: NoSchedule
        key: node-role.kubernetes.io/infra
      - effect: NoSchedule
        key: node-role.kubernetes.io/logging
  tenants:
    mode: openshift-logging
    openshift:
      adminGroups:
        - admins
```

> 💡 **Destaques:**
>
> * Este manifest define a instância do `LokiStack` que utilizará o bucket configurado anteriormente.
> * O bloco `limits.global` foi ajustado para suportar alto volume de logs.
> * O campo `storage.secret` referencia as credenciais do bucket (`lokistack-s3`), enquanto `tls.caName` aponta para o `ConfigMap` contendo a CA.
> * `storageClassName: vsphere-csi` integra-se diretamente ao backend do vSphere.
> * taint e tolerations foi aplicado conforme o machineset de exemplo desse artigo

---

### 3.6. UIPlugin

Crie o recurso `UIPlugin` para integrar a visualização do LokiStack ao Console do OpenShift:

```yaml
apiVersion: observability.openshift.io/v1alpha1
kind: UIPlugin
metadata:
  annotations:
    argocd.argoproj.io/sync-wave: "5"
    argocd.argoproj.io/sync-options: SkipDryRunOnMissingResource=true
  name: logging
spec:
  logging:
    lokiStack:
      name: lokistack
  type: Logging
```

> **Resultado:**
> Após a criação deste recurso, a aba *Logs* na Console Web do OpenShift passará a utilizar o Loki como backend de consulta, substituindo o Kibana.

---

### 3.5. ClusterLogForwarder CR

```yaml
apiVersion: observability.openshift.io/v1
kind: ClusterLogForwarder
metadata:
  annotations:
    argocd.argoproj.io/sync-wave: "5"
    argocd.argoproj.io/sync-options: SkipDryRunOnMissingResource=true
  name: lokistack-collector
  namespace: openshift-logging
spec:
  managementState: Managed
  collector:
    resources:
      limits:
        cpu: 2000m
        memory: 4Gi
      requests:
        cpu: 100m
        memory: 2Gi
    tolerations:
    - operator: Exists
  serviceAccount:
    name: logging-collector

  # --- Inputs ---
  inputs:
  - name: application
    type: application
    application: {}
  - name: infrastructure
    type: infrastructure
    infrastructure: {}
  - name: audit
    type: audit
    audit: {}

  # --- Filters ---
  filters:
  - name: app-filter-logs
    type: drop 
    drop: 
    - test: 
      - field: .message
        matches: "(?i)DEBUG"
      - field: .level
        matches: "debug" 
  
  - name: detect-multi-line
    type: detectMultilineException
  
  - name: audit-filter-logs
    type: kubeAPIAudit
    kubeAPIAudit:
      rules:
      - level: None
        users:
        - system:apiserver
        - system:kube-controller-manager
        - system:serviceaccount:kube-system:generic-garbage-collector
        - system:serviceaccount:open-cluster-management-agent-addon:application-manager
        - system:serviceaccount:open-cluster-management-agent-addon:cert-policy-controller-sa
        - system:serviceaccount:open-cluster-management-agent-addon:iam-policy-controller-sa
        - system:serviceaccount:open-cluster-management-agent-addon:klusterlet-addon-search
        - system:serviceaccount:open-cluster-management-agent-addon:klusterlet-addon-workmgr
        - system:serviceaccount:open-cluster-management-agent:klusterlet
        - system:serviceaccount:open-cluster-management-agent:klusterlet-registration-sa
        - system:serviceaccount:open-cluster-management-agent:klusterlet-work-sa
        - system:serviceaccount:openshift-apiserver:openshift-apiserver-sa
        - system:serviceaccount:openshift-apiserver-operator:openshift-apiserver-operator
        - system:serviceaccount:openshift-authentication:oauth-openshift
        - system:serviceaccount:openshift-authentication-operator:authentication-operator
        - system:serviceaccount:openshift-cloud-controller-manager:cloud-controller-manager
        - system:serviceaccount:openshift-cloud-controller-manager-operator:cluster-cloud-controller-manager
        - system:serviceaccount:openshift-cluster-csi-drivers:vmware-vsphere-csi-driver-node-sa
        - system:serviceaccount:openshift-cluster-csi-drivers:vmware-vsphere-csi-driver-operator
        - system:serviceaccount:openshift-cluster-observability-operator:observability-operator-sa
        - system:serviceaccount:openshift-cluster-storage-operator:cluster-storage-operator
        - system:serviceaccount:openshift-cluster-storage-operator:csi-snapshot-controller-operator
        - system:serviceaccount:openshift-cluster-version:default
        - system:serviceaccount:openshift-console-operator:console-operator
        - system:serviceaccount:openshift-controller-manager:openshift-controller-manager-sa
        - system:serviceaccount:openshift-controller-manager-operator:openshift-controller-manager-operator
        - system:serviceaccount:openshift-dns:dns
        - system:serviceaccount:openshift-dns-operator:dns-operator
        - system:serviceaccount:openshift-etcd-operator:etcd-operator
        - system:serviceaccount:openshift-gitops-operator:openshift-gitops-operator-controller-manager
        - system:serviceaccount:openshift-image-registry:cluster-image-registry-operator
        - system:serviceaccount:openshift-infra:serviceaccount-pull-secrets-controller
        - system:serviceaccount:openshift-ingress-operator:ingress-operator
        - system:serviceaccount:openshift-insights:operator
        - system:serviceaccount:openshift-kube-apiserver:check-endpoints
        - system:serviceaccount:openshift-kube-apiserver:localhost-recovery-client
        - system:serviceaccount:openshift-kube-apiserver-operator:kube-apiserver-operator
        - system:serviceaccount:openshift-kube-controller-manager:localhost-recovery-client
        - system:serviceaccount:openshift-kube-controller-manager-operator:kube-controller-manager-operator
        - system:serviceaccount:openshift-kube-scheduler:localhost-recovery-client
        - system:serviceaccount:openshift-kube-scheduler-operator:openshift-kube-scheduler-operator
        - system:serviceaccount:openshift-kube-storage-version-migrator-operator:kube-storage-version-migrator-operator
        - system:serviceaccount:openshift-logging:cluster-logging-operator
        - system:serviceaccount:openshift-machine-api:machine-api-controllers
        - system:serviceaccount:openshift-machine-api:machine-api-operator
        - system:serviceaccount:openshift-machine-config-operator:machine-config-controller
        - system:serviceaccount:openshift-machine-config-operator:machine-config-daemon
        - system:serviceaccount:openshift-machine-config-operator:machine-config-operator
        - system:serviceaccount:openshift-monitoring:cluster-monitoring-operator
        - system:serviceaccount:openshift-monitoring:prometheus-adapter
        - system:serviceaccount:openshift-network-operator:cluster-network-operator
        - system:serviceaccount:openshift-oauth-apiserver:oauth-apiserver-sa
        - system:serviceaccount:openshift-operator-lifecycle-manager:collect-profiles
        - system:serviceaccount:openshift-operator-lifecycle-manager:olm-operator-serviceaccount
        - system:serviceaccount:openshift-operators-redhat:loki-operator-controller-manager
        - system:serviceaccount:openshift-route-controller-manager:route-controller-manager-sa
        - system:serviceaccount:openshift-service-ca-operator:service-ca-operator
        - system:serviceaccount:psc-group-sync-operator:controller-manager
        - system:serviceaccount:psc-trident-operator:trident-operator
        verbs:
        - watch
        - get
        - list

      - level: Metadata
        resources:
        - group: ""
          resources:
          - secrets
          - configmaps
          - serviceaccounts

      - level: Metadata
        resources:
        - group: "rbac.authorization.k8s.io"
          resources:
          - roles
          - rolebindings
          - clusterroles
          - clusterrolebindings

      - level: Metadata
        resources:
        - group: "oauth.openshift.io"
          resources:
          - oauthclients

      - level: None
        verbs:
        - get
        - list
        - watch

  # --- Outputs ---
  outputs:
  - name: default-lokistack
    type: lokiStack
    lokiStack:
      authentication:
        token:
          from: serviceAccount
      target:
        name: lokistack
        namespace: openshift-logging
      tuning:
        maxRetryDuration: 30
        maxWrite: 20M
    rateLimit:
      maxRecordsPerSecond: 200
    tls:
      ca:
        configMapName: openshift-service-ca.crt
        key: service-ca.crt

  # --- Pipelines ---
  pipelines:
  - name: app-pipeline
    inputRefs: [application]
    filterRefs: [detect-multi-line, app-filter-logs]
    outputRefs: [default-lokistack]

  - name: infra-pipeline
    inputRefs: [infrastructure]
    filterRefs: [detect-multi-line]
    outputRefs: [default-lokistack]

  - name: audit-pipeline
    inputRefs: [audit]
    filterRefs: [detect-multi-line, audit-filter-logs]
    outputRefs: [default-lokistack]
```

## 4. Remover o Elasticsearch após o período de retenção

Após o tempo de retenção desejado:

```bash
oc -n openshift-logging delete kibana/kibana elasticsearch/elasticsearch
oc -n openshift-logging delete pvc -l logging-cluster=elasticsearch
```

---

## 5. Excluir o operador do Elasticsearch

Remova o CSV do **Elasticsearch Operator**

---

## Conclusão

Seguindo esses passos, você terá o **LokiStack** rodando no OpenShift como substituto completo do stack EFK.
Essa abordagem **simplifica a operação**, **melhora o desempenho** e **aumenta a escalabilidade** em ambientes com grande volume de logs.

> **Verificação final**
>
> ```bash
> oc -n openshift-logging get pods
> oc -n openshift-logging logs <collector-pod> | head
> ```

Esses comandos ajudam a confirmar que o LokiStack está ativo e recebendo logs corretamente.
