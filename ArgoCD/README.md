# ArgoCD 구성

## ArgoCD 설치

[공식 문서](https://argo-cd.readthedocs.io/en/stable/)를 참고하여 설치를 진행한다.

```sh
kubectl create namespace argocd
curl https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml -o argo-cd.yaml
```

> namespace를 생성했지만 사용하지는 않았음. 확인필요

바로 kubernetes에 적용해도 되지만 편한 접속을 위하여 service부분을 수정한다.

```yaml
apiVersion: v1
kind: Service
metadata:
  labels:
    app.kubernetes.io/component: server
    app.kubernetes.io/name: argocd-server
    app.kubernetes.io/part-of: argocd
  name: argocd-server
spec:
  type: NodePort
  ports:
  - name: http
    port: 80
    protocol: TCP
    targetPort: 8080
  - name: https
    port: 443
    protocol: TCP
    targetPort: 8080
  selector:
    app.kubernetes.io/name: argocd-server
```

`type: NodePort`를 추가하여 접속을 용이하게 한다.

```sh
kubectl apply -f argo-cd.yaml
# admin 계정의 password를 알아내는 코드
kubectl get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d
```
