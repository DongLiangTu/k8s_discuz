



## **Deploy the Discuz forum using K8S and use the Nginx proxy to access**

### **system structure drawing：**

​            ![img](https://wdoc-76491.picgzc.qpic.cn/MTY4ODg1MDk1MjI1NTg1NQ_948222_9L6CUv2WuVNdLTjJ_1626954296)            

### **Pull images：**

```
docker pull mysql:5.7.22
docker pull skyzhou/docker-discuz
```

### **Deploy mysql service：**

```
kubectl apply -f mysql-dp.yml
kubectl apply -f mysql-svc.yml
```

#### **Verify Mysql：**

- Obtain the mysql SVC, Record the cluster-ip (**172.16.254.96**) of mysql-service, and configure this IP  when creating a link with discuz SVC  

```
kubectl get svc -o wide
```

​            ![img](https://wdoc-76491.picgzc.qpic.cn/MTY4ODg1MDk1MjI1NTg1NQ_277283_holsXLC7IQ-D4KYn_1626858207)            

- get mysql pods name

```
kubectl get pods -o wide
```

- sign in mysql

```
kubectl exec -it mysql-5f87f4495c-9stsn  -- mysql -uroot -proot
```

​            ![img](https://wdoc-76491.picgzc.qpic.cn/MTY4ODg1MDk1MjI1NTg1NQ_831926_68YFjoYK-H6D8izs_1626858435)            



#### **Discuz deployment**

-  \# vim discuz-dp.yml

**Need to modify MYSQL_PORT_3306_TCP corresponding key value to ${mysql-service LUSTER-IP:port}**



```
apiVersion: apps/v1         # apiserver的版本
kind: Deployment            # 副本控制器deployment，管理pod和RS
metadata:
  name: discuz              # deployment的名称，全局唯一
spec:
  replicas: 1               # Pod副本期待数量
  selector:
    matchLabels:            # 定义RS的标签
      app: discuz           # 符合目标的Pod拥有此标签
  strategy:                 # 定义升级的策略
    type: RollingUpdate     # 滚动升级，逐步替换的策略
  template:                 # 根据此模板创建Pod的副本（实例）
    metadata:
      labels:
        app: discuz         # Pod副本的标签，对应RS的Selector
    spec:
      containers:           # Pod里容器的定义部分
        - name: discuz      # 容器的名称
          image: skyzhou/docker-discuz
          ports:
            - containerPort: 80
          env:              # 写入到容器内的环境容量
            - name: MYSQL_PORT_3306_TCP
              value: **172.16.254.96:3306**
            - name: DISCUZ_DB_PASSWORD
              value: root
```



#### **Deployment Discuz**

```
kubectl apply -f discuz-dp.yml
kubectl apply -f discuz-svc.yml
```



#### **Verify Discuz：**

```
kubectl get deployment -o wide
kubectl get svc -o wide
```

- Access to the cluster IP:30080

​            ![img](https://wdoc-76491.picgzc.qpic.cn/MTY4ODg1MDk1MjI1NTg1NQ_588843_1arpj8fKg9EpeZ01_1626858913)            



### **Deploy Nginx：**

- pull images

```
docker pull nginx:stable
```



  \# vim nginx.conf  Enter the following configuration to configure the **URL for forwarding on proxy_pass **

```
user  nginx;
worker_processes  auto;

error_log  /var/log/nginx/error.log notice;
pid        /var/run/nginx.pid;


events {
    worker_connections  1024;
}


http {
    include       /etc/nginx/mime.types;
    default_type  application/octet-stream;

    log_format  main  '$remote_addr - $remote_user [$time_local] "$request" '
                    '$status $body_bytes_sent "$http_referer" '
                    '"$http_user_agent" "$http_x_forwarded_for"';
    access_log  /var/log/nginx/access.log  main;
    sendfile        on;
    #tcp_nopush     on;
    keepalive_timeout  65;
    #gzip  on;
    server {
                listen       80;
                server_name  localhost;
                location / {
                        proxy_pass **http://43.128.70.95:30080/install/;**
                }
            }
    include /etc/nginx/conf.d/*.conf;
}
```

- create configmap

```
kubectl create configmap nginx-conf --from-file=./nginx.conf
```

- nginx deployment svc

```
kubectl apply -f nginx-dp.yml
kubectl apply -f nginx-svc.yml
```



### **Verify Nginx proxy access：**

- Access to the cluster IP:30180

​            ![img](https://wdoc-76491.picgzc.qpic.cn/MTY4ODg1MDk1MjI1NTg1NQ_800638_g61j3FSty8eF7MuQ_1626875803)            

- Follow the prompts to install Discuz:

​            ![img](https://wdoc-76491.picgzc.qpic.cn/MTY4ODg1MDk1MjI1NTg1NQ_843386_a3FSCzqulw-0DTEL_1626858941)            

​            ![img](https://wdoc-76491.picgzc.qpic.cn/MTY4ODg1MDk1MjI1NTg1NQ_596058_nkhtsEpvv-Mf_2d8_1626859011)            

