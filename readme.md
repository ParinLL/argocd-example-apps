# 安裝Argo

## 安裝Argo CD

```
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

## 下載Argo CD CLI

```
VERSION=$(curl --silent "https://api.github.com/repos/argoproj/argo-cd/releases/latest" | grep '"tag_name"' | sed -E 's/.*"([^"]+)".*/\1/')
curl -sSL -o /usr/local/bin/argocd https://github.com/argoproj/argo-cd/releases/download/$VERSION/argocd-linux-amd64
chmod +x /usr/local/bin/argocd
```

## ingress

```
cat <<EOF | kubectl apply -f -
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: argocd-server-ingress
  annotations:
    kubernetes.io/ingress.class: "nginx"
    nginx.ingress.kubernetes.io/backend-protocol: "HTTPS"
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
    nginx.ingress.kubernetes.io/ssl-passthrough: "true"
spec:
  rules:
  - host: argo.tradevan.com.tw
  - http:
      paths:
      - path: /
        backend:
          serviceName: argocd-server
          servicePort: https
EOF


```



## 取得ArgoCD admin密碼

```
kubectl get pods -n argocd -l app.kubernetes.io/name=argocd-server -o name | cut -d'/' -f 2

## 出現的東西即是密碼
argocd-server-656f9b895b-bfjvw
```



## 登入並修改 密碼

```
argocd login argo.tradevan.com.tw
### 這裡會要求輸入帳密
Username: admin
Password: 

## 變更密碼
argocd account update-password
*** Enter current password: 
*** Enter new password: 
*** Confirm new password: 
```



## 修改argo 所使用的sa權限

+ 預設值只能在default 的namespace 裡面活動
+ 執行下面指令會將現在的kube config 裡關於 kube-system 的權限綁定給 此sa使用 (如果你正在登入的權限為cluster-admin, 此時argo的sa 也會同樣取得此ClusterRole)

```
argocd cluster add
```



連線到UI: https://argo.tradevan.com.tw

----



# 測試流程

## 複製git 範例

+ 再登入自己的github 
+ 連線到下面的範例網址  https://github.com/harryliu123/argocd-example-apps
+ 按下右上角的 Fork 取得網址 https://github.com/<yourname>/argocd-example-apps
+ 修改



## 透過cli佈署測試project (當然也可以透過UI執行)

+ path 為 guestbook
+ 由於 argo 所佈署的k8s 和 guestbook 為同一座k8s 所以 kube api為https://kubernetes.default.svc
+ 將服務佈署在 default的 namespace上
+ 修改 guestbook/guestbook-ui-ingress.yaml 裡面的網址: guestbook.tradevan-rd.com

```
argocd app create guestbook --repo  https://github.com/<yourname>/argocd-example-apps --path guestbook --dest-server https://kubernetes.default.svc --dest-namespace default
```

連線 http://guestbook.tradevan-rd.com  即可看到網頁



## 在Git 上變更狀態看是否有改變

### 查看 目前狀態

```
$ kubectl get po
NAME                            READY   STATUS    RESTARTS   AGE
guestbook-ui-85c9c5f9cb-fdc97   1/1     Running   0          14h
```

### git 上修改guestbook-ui-deployment.yaml

```
...
spec:
  replicas: 3  --> 原本1改為3
...

```

### 再查看狀態

```
$ kubectl get po
NAME                            READY   STATUS              RESTARTS   AGE
guestbook-ui-85c9c5f9cb-7vw7w   0/1     ContainerCreating   0          4s
guestbook-ui-85c9c5f9cb-fdc97   1/1     Running             0          16h
guestbook-ui-85c9c5f9cb-kgqrh   0/1     ContainerCreating   0          4s
```



