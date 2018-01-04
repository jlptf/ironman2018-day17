## Day 17 - Secret

### 本日共賞

* Secret

### 希望你知道

* [與 k8s 溝通: kubectl](https://ithelp.ithome.com.tw/articles/10193502)
* [yaml](https://ithelp.ithome.com.tw/articles/10193509)

<br/>

#### Secret

昨天我們提到了 [ConfigMap](https://ithelp.ithome.com.tw/articles/10193935)，Secret 儲存的資料是較為敏感的內容 (密碼、金鑰等等)，而 ConfigMap 則是儲存像設定之類較不敏感的資料。Secret 概念上類似 ConfigMap，也是希望將資料與應用程式解耦，除此之外，針對較敏感的資料也能有多一層保護而不隨便曝露。

Secret 有三種類型：

1. `Service Account`：由 k8s 自動建立並掛載到 Pod，用來存取 k8s API 使用，你可以在 `/run/secrets/kubernetes.io/serviceaccount` 目錄中找到
2. `Opaque`：以 base64 編碼的 Secret，用來儲存 密碼、金鑰等等
3. `docker-registry`：如果映像檔是放在私有的 Registry，就需要使用這種類型的 Secret

> 你可以嘗試利用 [Day 14 - Volume (1)](https://ithelp.ithome.com.tw/articles/10193546) 提到過的 `kubectl exec` 指令進入 Pod 查看 `/run/secrets/kubernetes.io/serviceaccount` 目錄下的 Secret 物件

**Opaque**

假設你的應用程式需要存取資料庫，這時候你會需要帳號 (username) 與密碼 (password)，所以我們可以建立兩個檔案

```bash
$ echo -n "admin" > username.txt  <=== -n:不換行輸出

$ echo -n "mypassword" > password.txt
```


接著建立 Secret 物件

```bash
$ kubectl create secret generic db-user-pass \
--from-file=./username.txt \
--from-file=./password.txt
secret "db-user-pass" created
```

查看一下 Secret 的內容

```bash
$ kubectl get secrets   <=== 利用 get 查看一下
NAME                  TYPE                                  DATA      AGE
db-user-pass          Opaque                                2         1m

$ kubectl describe secrets/db-user-pass   <=== 利用 describe 查看一下
Name:         db-user-pass
Namespace:    default
Labels:       <none>
Annotations:  <none>

Type:  Opaque

Data
====
password.txt:  10 bytes
username.txt:  5 bytes
```

這裡可以發現，無論你是使用 `get` 或是 `describe` 都不會顯示 `username` 與 `password` 的內容。

如果想要使用 yaml 的方式部署 Secret 請先將內容編碼 (base64) 後再填入 yaml 檔

```bash
$ echo -n "admin" | base64   <=== 指定 base64 編碼
YWRtaW4=   <=== yaml 需要填入這個

$ echo -n "mypassword" | base64
bXlwYXNzd29yZA==
```

然後編寫 yaml

```
# secret.yaml

---
apiVersion: v1
kind: Secret
metadata:
  name: mysecret
type: Opaque
data:
  username: YWRtaW4=    <=== 這裡要記得使用 base64 編碼後的資料
  password: bXlwYXNzd29yZA==   <=== 這裡要記得使用 base64 編碼後的資料
```

然後部署到 k8s

```bash
$ kubectl apply -f secret.yaml
secret "mysecret" created
```

建立好 Secret 物件後，要如何使用呢？你可以使用 [Day 14 - Volume (1)](https://ithelp.ithome.com.tw/articles/10193546) 提到的 Volume 掛載到 Pod 中使用。

```
# secret.pod.yaml

---
apiVersion: v1
kind: Pod
metadata:
  name: secret-nginx
spec:
  containers:
  - name: nginx
    image: nginx
    volumeMounts:
    - mountPath: /tmp/conf  <=== 將綁定 mysecret 的 Volume 掛載到 /tmp/conf 目錄下
      name: secret-volume
    env:                    <=== 將綁定的 mysecret 以環境變數的方式掛載
    - name: DB_PASSWORD    
      valueFrom:
        secretKeyRef:
          name: mysecret   
          key: password     <=== 指定綁定哪個 key
  volumes:
  - name: secret-volume   
    secret:
      secretName: mysecret   <=== 指定使用名為 mysecret 的 Secret 物件
```

部署 `secret-nginx` 到 k8s

```bash
$ kubectl apply -f secret.pod.yaml
pod "secret-nginx" created
```

這裡示範了兩種掛載 Secret 的方式：

**1. 將 Secret 指定到環境變數 `DB_PASSWORD` 中**

```bash
$ kubectl exec -it secret-nginx bash   <=== 連到 secret-nginx Pod

$ env | grep DB_PASSWORD    <=== 檢查環境變數 DB_PASSWORD
DB_PASSWORD=mypassword 
```

**2. 將 Secret 以 Volume 形式掛載到 `/tmp/conf`**

```bash
$ kubectl exec -it secret-nginx bash   <=== 連到 secret-nginx Pod
$ ls /tmp/conf   <=== 查詢是否有 username 與 password
password  username
```

因此，如果應用程式需要使用到較敏感的資料，我們就可以利用 Secret 儲存這些資料，等到 Pod 運行之後再將這些資料掛到指定目錄下。這樣做會有幾個好處：

* 避免敏感資料被任意讀取
* 可由管理者統一管理
* 可重複使用

**docker-registry**

如果你要使用的映像檔不是放在公開的 registry，這時候你就需要使用 `docker-registry` 類型的 Secret 儲存帳號跟密碼，以便 k8s 可以順利下載映像檔。

使用的格式如下

```bash
kubectl create secret docker-registry <name> \
--docker-server=<your-registry-server> \
--docker-username=<your-name> \
--docker-password=<your-pword> \
--docker-email=<your-email>
```

* `docker-registry`：指定為 `docker-registry` 形態的 Secret
* `--docker-server`：私有 registry server 位置
* `--docker-username`：Docker 帳號
* `--docker-password`：Docker 密碼
* `--docker-email`：Docker Email

> 這裡會使用 Docker 私有的映像檔做示範，建議可以去申請一個 Docker 帳號，每個帳號可以有一個免費的私有映像檔。因為，各位總不會這麼殘忍要我提供密碼吧...
> 
> 這裡先假設大家已經建立好私有的映像檔

因此，我以 [Docker Hub](hub.docker.com) 為例，建立一個名為 `regsecret` 的 Secret 物件。

```bash
$ kubectl create secret docker-registry regsecret \
--docker-server=https://index.docker.io/v1/ \
--docker-username=<換成你的> \
--docker-password=<換成你的> \
--docker-email=<換成你的>
secret "regsecret" created
```

我在 [Docker Hub](hub.docker.com) 建立了一個私有映像檔 `jlptf/privateapp`，接著修改 yaml 的內容

```
# private-pod.yaml

---
apiVersion: v1
kind: Pod
metadata:
  name: private-reg
spec:
  containers:
  - name: private-reg
    image: jlptf/privateapp   <=== 指定使用私有映像檔
    imagePullPolicy: Always
  imagePullSecrets:   <=== 下載映像檔時套用先前建立的 Secret 物件
  - name: regsecret
```

然後部署到 k8s 

```bash
$ kubectl apply -f private-pod.yaml
pod "private-reg" created

$ kubectl get pods
NAME           READY     STATUS    RESTARTS   AGE
private-reg    1/1       Running   0          18m
```

到這裡就表示一切進行順利，k8s 也能夠取得私有映像檔進行配置。所以聰明的各位應該能夠聯想到，如果需要存取同一個私有的 registry，其他部署也可以使用同一個 Secret。

本文與部署檔案同步發表於 [https://jlptf.github.io/ironman2018-day17/](https://jlptf.github.io/ironman2018-day17/)