TEST
===

.. autosummary::
   :toctree: generated

   # cert-manager 管理憑證

* 安裝好 cert-manager 之後，我們要來設定簽發 TLS 憑證的組態，而這邊有兩種類別，我們這個範例使用 ClusterIssuer
  - Issuer：作用範圍只在某個 K8S Namespace 內
  - ClusterIssuer：作用範圍為整個 K8S Cluster
  - Certificate : 物件表示 TLS/SSL 證書。它包含證書的 PEM 編碼格式、私鑰、以及其他 metadata。
* 這個設定主要是要讓 cert-manager 知道所要使用的 CA 是什麼 (像這邊就是使用 Let’s Encrypt)，還有要管理的域名以及驗證的方法，設定好這些之後，下一個步驟要申請 Certificate 的時候就可以直接引用它


## 使用 cert-manager SelfSigned 部屬 ClusterIssuer
```
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: selfsigned-cluster-issuer
spec:
  selfSigned: {}
```
```
$ kubectl get clusterissuers -o wide selfsigned-cluster-issuer
NAME                        READY   STATUS   AGE
selfsigned-cluster-issuer   True             10s
```
## 部屬 Certificate
```
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: selfsigned-ca
  namespace: default
spec:
  isCA: true
  commonName: selfsigned-ca
  secretName: root-secret
  privateKey:
    algorithm: ECDSA
    size: 256
  issuerRef:
    name: selfsigned-cluster-issuer
    kind: ClusterIssuer
    group: cert-manager.io
```
```
$ kubectl get certificate
NAME            READY   SECRET        AGE
selfsigned-ca   True    root-secret   6s
```
* 檢查 cert-manager 建的 secret
```
$ kubectl get secret root-secret
NAME          TYPE                DATA   AGE
root-secret   kubernetes.io/tls   3      8m27s
```

```
$ kubectl get certificaterequest
NAME                  APPROVED   DENIED   READY   ISSUER                      REQUESTOR                                         AGE
selfsigned-ca-xxwg4   True                True    selfsigned-cluster-issuer   system:serviceaccount:cert-manager:cert-manager   61s
```

## 使用 cert-manager 建立的憑證建立 ingress
* 建立測試用 nginx web
```
$ kubectl create deploy web --image=nginx --port=80
```
```
$ kubectl expose deploy web --name=svc-web --target-port=80 --port=80
```
* 檢查 ingressclass 名稱
* 需要 dns 解析 `test.cooloo9871.com` 這個位置
```
$ kubectl get ingressClass
NAME    CONTROLLER             PARAMETERS   AGE
nginx   k8s.io/ingress-nginx   <none>       4d23h
```
```
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: tls-nginx-ingress
spec:
  ingressClassName: nginx
  tls:
  - hosts:
      - test.cooloo9871.com
    secretName: root-secret
  rules:
  - host: test.cooloo9871.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: svc-web  
            port:
              number: 80
```


## 將 cert-manager 簽的憑證匯入到 client 瀏覽器

