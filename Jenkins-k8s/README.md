# Jenkins 구성

[공식문서](https://www.jenkins.io/doc/book/installing/kubernetes/)를 참고하여 Kubernetes에 Jenkins를 설치한다.

Helm을 이용하지 않고 menifest file을 이용하여 설치를 진행한다.

## Jenkins Kubernetes Manifest Files

먼저 manifest file이 있는 repository를 clone하여 준비한다.

```sh
git clone https://github.com/scriptcamp/kubernetes-jenkins
```

5개의 yaml파일이 존재한다.

- deployment.yaml
- namespace.yaml
- service.yaml
- serviceAccount.yaml
- volume.yaml

### namespace 생성

Jenkins를 위한 Namespace를 생성한다.

```sh
kubectl create namespace devops-tools
```

### service account 생성

`serviceAccount.yaml` 을 적용한다.

```yaml
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: jenkins-admin
rules:
  - apiGroups: [""]
    resources: ["*"]
    verbs: ["*"]

---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: jenkins-admin
  namespace: devops-tools

---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: jenkins-admin
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: jenkins-admin
subjects:
- kind: ServiceAccount
  name: jenkins-admin
```

```sh
kubectl apply -f serviceAccount.yaml
```

> role 과 account를 생성하고 부작한다.

### storage 생성

`volume.yaml`을 적용한다.

```yaml
kind: StorageClass
apiVersion: storage.k8s.io/v1
metadata:
  name: local-storage
provisioner: kubernetes.io/no-provisioner
volumeBindingMode: WaitForFirstConsumer

---
apiVersion: v1
kind: PersistentVolume
metadata:
  name: jenkins-pv-volume
  labels:
    type: local
spec:
  storageClassName: local-storage
  claimRef:
    name: jenkins-pv-claim
    namespace: devops-tools
  capacity:
    storage: 10Gi
  accessModes:
    - ReadWriteOnce
  local:
    path: /mnt
  nodeAffinity:
    required:
      nodeSelectorTerms:
      - matchExpressions:
        - key: kubernetes.io/hostname
          operator: In
          values:
          - worker-node01

---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: jenkins-pv-claim
  namespace: devops-tools
spec:
  storageClassName: local-storage
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 3Gi
```

PersistentVolume의 `spec` > `nodeAffinity` > `required` > `nodeSelectorTerms` > `matchExpressions` > `values` 의 `worker-node01`은 클러스터의 node들 중 worker node의 이름으로 대체한다.

`apply`, `create` 둘다 상관없지만 새로운 리소스를 생성할 때에는 `create`를 주로 사용한다.

```sh
kubectl apply -f volume.yaml
kubectl create -f volume.yaml
```

> worker-node01의 /mnt에 10gb용량의 PV를 생성한다.

### deploy

`deployment.yaml`을 적용한다.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: jenkins
  namespace: devops-tools
spec:
  replicas: 1
  selector:
    matchLabels:
      app: jenkins-server
  template:
    metadata:
      labels:
        app: jenkins-server
    spec:
      securityContext:
            fsGroup: 1000
            runAsUser: 1000
      serviceAccountName: jenkins-admin
      containers:
        - name: jenkins
          image: jenkins/jenkins:lts
          resources:
            limits:
              memory: "2Gi"
              cpu: "1000m"
            requests:
              memory: "500Mi"
              cpu: "500m"
          ports:
            - name: httpport
              containerPort: 8080
            - name: jnlpport
              containerPort: 50000
          livenessProbe:
            httpGet:
              path: "/login"
              port: 8080
            initialDelaySeconds: 90
            periodSeconds: 10
            timeoutSeconds: 5
            failureThreshold: 5
          readinessProbe:
            httpGet:
              path: "/login"
              port: 8080
            initialDelaySeconds: 60
            periodSeconds: 10
            timeoutSeconds: 5
            failureThreshold: 3
          volumeMounts:
            - name: jenkins-data
              mountPath: /var/jenkins_home
      volumes:
        - name: jenkins-data
          persistentVolumeClaim:
              claimName: jenkins-pv-claim
```

```sh
kubectl apply -f deployment.yaml
kubectl get deployments -n devops-tools
kubectl describe deployments --namespace=devops-tools
```

### service 생성

`service.yaml`을 적용한다.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: jenkins-service
  namespace: devops-tools
  annotations:
      prometheus.io/scrape: 'true'
      prometheus.io/path:   /
      prometheus.io/port:   '8080'
spec:
  selector:
    app: jenkins-server
  type: NodePort
  ports:
    - port: 8080
      targetPort: 8080
      nodePort: 32000
```

```sh
kubectl apply -f service.yaml
kubectl get pods --namespace=devops-tools
```

정상적으로 동작할 경우 `http://nodeip:32000`으로 접속이 가능하다.
이후 세팅은 ubuntu 에 설치할 때와 같다.

> 초기 비밀번호는 직접 들어가서 확인하거나 `kubectl get pods -n devops-tools`를 통해 `pod name`을 알아내고 `kubectl logs podname -n devops-tools`을 통해 로그를 확인하여 알아낼 수 있다.
