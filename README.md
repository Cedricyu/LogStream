# Distributed Log Collection and Visualization System

## Pre-installation

### Install Docker (Linux)
```bash
sudo apt update  
sudo apt install -y docker.io  
sudo systemctl start docker  
sudo systemctl enable docker
```

### Verify Docker
```bash
docker --version
```
> You should see the Docker version info if the installation was successful.

### Install k3d (Lightweight Kubernetes)
```bash
curl -s https://raw.githubusercontent.com/k3d-io/k3d/main/install.sh | bash
```

### Verify k3d
```bash
k3d --version
```

### Create Cluster
```bash
k3d create my-cluster-0 --port 80:80
```

## Deploy Frontend Build Folder
```bash
docker cp ./dist k3d-mycluster-server-0:/var/lib/rancher/k3s/storage/[nginx-pvc-path]
```

## Log Collection: Filebeat + Logstash + Elasticsearch + Kibana (ELK Stack)

### Filebeat ConfigMap
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: filebeat-config
data:
  filebeat.yml: |
    filebeat.inputs:
    - type: log
      enabled: true
      paths:
        - /var/log/nginx/*.log
    output.logstash:
      hosts: ["<LOGSTASH_HOST>:5044"]
```

### Apply ConfigMap
```bash
kubectl apply -f filebeat-config.yaml
```

## Nginx + Filebeat Deployment
Includes:
- Frontend (nginx)
- Log collector (filebeat)
- Shared log volume

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
spec:
  replicas: 2
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      serviceAccountName: filebeat
      containers:
      - name: nginx
        image: nginx:latest
        ports:
        - containerPort: 80
        volumeMounts:
        - name: nginx-html
          mountPath: /usr/share/nginx/html
          subPath: dist
        - name: nginx-logs
          mountPath: /var/log/nginx
      - name: filebeat
        image: docker.elastic.co/beats/filebeat:8.10.0
        args: ["-e", "-c", "/usr/share/filebeat/filebeat.yml"]
        env:
        - name: NODE_NAME
          valueFrom:
            fieldRef:
              fieldPath: spec.nodeName
        volumeMounts:
        - name: filebeat-config
          mountPath: /usr/share/filebeat/filebeat.yml
          subPath: filebeat.yml
        - name: nginx-logs
          mountPath: /var/log/nginx
      volumes:
      - name: nginx-html
        persistentVolumeClaim:
          claimName: nginx-pvc
      - name: nginx-logs
        emptyDir: {}
      - name: filebeat-config
        configMap:
          name: filebeat-config
```

## Logstash Pipeline: Threat Detection
Filters:
- XSS
- Path Traversal
- LFI (Local File Inclusion)
- RFI (Remote File Inclusion)
- HTTP Flood

Outputs:
- Security events → Elasticsearch (security-events index)
- Default → Nginx logs index

> Passwords and sensitive IPs should be securely managed using Secrets.

## Elastic Stack Deployment

### Elasticsearch Persistent Volume
```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: elasticsearch-pv
spec:
  capacity:
    storage: 10Gi
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  volumeMode: Filesystem
  storageClassName: local-path
  hostPath:
    path: /mnt/data/elasticsearch
    type: DirectoryOrCreate
```

### Elasticsearch Stateful Deployment
Contains:
- Auth credentials
- Persistent data mount
- Exposed ports: 9200, 9300

### Kibana Deployment
Connected to Elasticsearch.  
Make sure to set:
- `ELASTICSEARCH_HOSTS`
- `ELASTICSEARCH_USERNAME`
- `ELASTICSEARCH_PASSWORD`
- `XPACK_ENCRYPTED_SAVED_OBJECTS_ENCRYPTION_KEY`

## Monitoring Access
Kibana Dashboard:  
> Visit `http://<KIBANA_HOST>:5601/` to verify UI and visual logs.