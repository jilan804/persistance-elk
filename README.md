# persistance-elk

---
apiVersion: v1
kind: ConfigMap
metadata:
  name: filebeat-config
  namespace: kube-system
  labels:
    k8s-app: filebeat
    kubernetes.io/cluster-service: "true"
data:
  filebeat.yml: |-
    filebeat.config:
      prospectors:
        # Mounted `filebeat-prospectors` configmap:
        path: ${path.config}/prospectors.d/*.yml
        # Reload prospectors configs as they change:
        reload.enabled: false
      modules:
        path: ${path.config}/modules.d/*.yml
        # Reload module configs as they change:
        reload.enabled: false

    processors:
      - add_cloud_metadata:

    cloud.id: ${ELASTIC_CLOUD_ID}
    cloud.auth: ${ELASTIC_CLOUD_AUTH}

    output.elasticsearch:
      hosts: ['${ELASTICSEARCH_HOST:elasticsearch}:${ELASTICSEARCH_PORT:9200}']
      username: ${ELASTICSEARCH_USERNAME}
      password: ${ELASTICSEARCH_PASSWORD}
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: filebeat-prospectors
  namespace: kube-system
  labels:
    k8s-app: filebeat
    kubernetes.io/cluster-service: "true"
data:
  kubernetes.yml: |-
    - type: log
      paths:
        - /var/lib/docker/containers/*/*.log
      json.message_key: log
      json.keys_under_root: true
      processors:
        - add_kubernetes_metadata:
            in_cluster: true
            namespace: ${POD_NAMESPACE}
---
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: filebeat
  namespace: kube-system
  labels:
    k8s-app: filebeat
    kubernetes.io/cluster-service: "true"
spec:
  selector:
    matchLabels:
      k8s-app: filebeat
  template:
    metadata:
      labels:
        k8s-app: filebeat
        kubernetes.io/cluster-service: "true"
    spec:
      serviceAccountName: filebeat
      terminationGracePeriodSeconds: 30
      containers:
      - name: filebeat
        image: docker.elastic.co/beats/filebeat:6.2.4
        args: [
          "-c", "/etc/filebeat.yml",
          "-e",
        ]
        env:
        - name: ELASTICSEARCH_HOST
          value: elasticsearch-logging.elk.svc.cluster.local
        - name: ELASTICSEARCH_PORT
          value: "9200"
        - name: ELASTICSEARCH_USERNAME
          value: elastic
        - name: ELASTICSEARCH_PASSWORD
          value: changeme
        - name: ELASTIC_CLOUD_ID
          value:
        - name: ELASTIC_CLOUD_AUTH
          value:
        - name: POD_NAMESPACE
          valueFrom:
            fieldRef:
              fieldPath: metadata.namespace
        securityContext:
          runAsUser: 0
        resources:
          limits:
            memory: 200Mi
          requests:
            cpu: 100m
            memory: 100Mi
        volumeMounts:
        - name: config
          mountPath: /etc/filebeat.yml
          readOnly: true
          subPath: filebeat.yml
        - name: prospectors
          mountPath: /usr/share/filebeat/prospectors.d
          readOnly: true
        - name: data
          mountPath: /usr/share/filebeat/data
        - name: varlibdockercontainers
          mountPath: /var/lib/docker/containers
          readOnly: true
      volumes:
      - name: config
        configMap:
          defaultMode: 0600
          name: filebeat-config
      - name: varlibdockercontainers
        hostPath:
          path: /var/lib/docker/containers
      - name: prospectors
        configMap:
          defaultMode: 0600
          name: filebeat-prospectors
      - name: data
        emptyDir: {}
---
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: ClusterRoleBinding
metadata:
  name: filebeat
subjects:
- kind: ServiceAccount
  name: filebeat
  namespace: kube-system
roleRef:
  kind: ClusterRole
  name: filebeat
  apiGroup: rbac.authorization.k8s.io
---
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: ClusterRole
metadata:
  name: filebeat
  labels:
    k8s-app: filebeat
rules:
- apiGroups: [""] # "" indicates the core API group
  resources:
  - namespaces
  - pods
  verbs:
  - get
  - watch
  - list
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: filebeat
  namespace: kube-system
  labels:
    k8s-app: filebeat

============================================================
application ingress:-
---
kind: Deployment
apiVersion: apps/v1
metadata:
  name: app-config
  namespace: logs
  labels:
    k8s-app: app
    kubernetes.io/cluster-service: "true"
spec:
  selector:
    matchLabels:
      k8s-app: app
  replicas: 2
  template:
    metadata:
      labels:
        k8s-app: app
        kubernetes.io/cluster-service: "true"
    spec:
      containers:
      - name: nginx-app
        image: 292090429255.dkr.ecr.ap-south-1.amazonaws.com/a.ysquaretechnology.com:latest
        ports:
        - containerPort: 80

---
kind: Service
apiVersion: v1
metadata:
  name: nginx-service
  namespace: logs
spec:
  selector:
    k8s-app: app
    kubernetes.io/cluster-service: "true"
  ports:
    - protocol: TCP
      port: 80
      targetPort: 80

---
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: ingress
  namespace: logs
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  rules:
  - host: b.example.com
    http:
      paths:
        - path:
          backend:
            serviceName: nginx-service
            servicePort: 80


