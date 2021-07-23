



## **Deploy the Discuz forum using K8S and access it using the Nginx proxy**

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
apiVersion: apps/v1         
kind: Deployment           
metadata:
  name: discuz              
spec:
  replicas: 1             
  selector:
    matchLabels:         
      app: discuz           
  strategy:                
    type: RollingUpdate     
  template:                
    metadata:
      labels:
        app: discuz        
    spec:
      containers:           
        - name: discuz      
          image: skyzhou/docker-discuz
          ports:
            - containerPort: 80
          env:             
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

