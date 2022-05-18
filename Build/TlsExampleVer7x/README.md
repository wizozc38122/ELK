# TLS 加密實作紀錄
> !! 只是個人實作紀錄，並非正確方式 !!
## 簡介
此範例為Kibana & Elasticsearch & Logstash & Filebeat之間TLS的加密設定


+ 測試環境
    + CentOS - 7.x
    + ELK - 7.16.2
    + 官方RPM安裝路徑(/etc/* , /usr/share/*/bin)

+ 測試資料
    ```sh
    # 共三台主機來代表Elasticserch Cluster
    # 沒有DNS自行註冊/etc/hosts
    # Kibane&Logstash&Filebeat用Node1暫時代替
    192.168.0.1 Node1.test.com
    192.168.0.2 Node2.test.com
    192.168.0.3 Node3.test.com
    ```

# 自簽憑證
由於自簽自己就是CA，要先建立自簽CA憑證再往下簽 

再以 `wildcard` 方式建立共用憑證，給予ELK各應用使用 

而內建ES就有 `elasticsearch-certutil`  這個指令來給你產生憑證 

當然你要用`openssl`等其他工具也可以

1. 產生自簽憑證
    ```sh
    # args => ca
    elasticsearch-certutil ca
    ```
    產生`elastic-stack-ca.p12`放到你存放憑證的路徑中
2. 簽發共用憑證
    ```sh
    # args => http
    elasticsearch-certutil http 
    => 是否使用已存在CA，選擇剛才建立的CA (elastic-stack-ca.p12) 
    => SAN 這是關鍵，請輸入*.test.com (wildcard)
    ```
    會產生2個目錄

    1. elastichsearch/ 內的 `http.p12` 就是Server用的
    2. kibana/ 內的 `elasticsearch-ca.pem` 就是給Client的信任憑證



# Elasticsearch Cluster
預設`Elasticsearch Cluster`之間溝通是用`9300` Port，這之間也要加密

使用前面建立的`http.p12`共用即可

1. 配置ES => transport.ssl
    ```yaml
    # vim  elasticsearch.yml 

    xpack.security.transport.ssl.enabled: true  
    xpack.security.transport.ssl.verification_mode: full  
    xpack.security.transport.ssl.keystore.path: /your_path/http.p12
    xpack.security.transport.ssl.truststore.path: /your_path/http.p12 
    ```

2. 檢查 `http://ES:9200/_nodes` 若仍有多節點代表成功

# Kibana 連接 Elasticsearch
預設`Elasticsearch`存取API Port為`9200`，對於`Kibana`來說，他要去存取`Elasticsearch`，`Elasticsearch`就是`Server`端

因此`Kibana`就是`Client`端，需要`Server`端證書的信任憑證

`Kibana`使用前面得到信任憑證的`elasticsearch-ca.pem `

`Elasticsearch`就使用`Server`端憑證`http.p12`

1. 配置ES => https.ssl
    ```yaml
    # vim  elasticsearch.yml 

    xpack.security.http.ssl.enabled: true  
    xpack.security.http.ssl.keystore.path: your_path/http.p12 
    xpack.security.http.ssl.truststore.path: your_path/http.p12 
    ```
2. 配置Kibana => 連接ES.ssl
    ```yaml
    # vim  kibana.yml

    elasticsearch.hosts: ["https://Node1.test.com:9200", "https://Node2.test.com:9200","https://Node3.test.com:9200"] 
    elasticsearch.ssl.certificateAuthorities: ["/your_path/elasticsearch-ca.pem"] 
    elasticsearch.ssl.verificationMode: full 
    ```
# Logstash 連接 Elasticsearch
這一步就是`Logstash`與`Elasticsearch`之間傳輸加密，與`Kibana`一樣都屬於`Client`的角色

所以拿Client那份 `elasticsearch-ca.pem` 信任憑證即可
1. Logstash Pipeline Config => output
    ```ruby
    output {  
        elasticsearch {  
            hosts => ["https://Node1.test.com:9200", "https://Node2.test.com:9200", "https://Node3.test.com:9200"]  
            index => "your_index"  
            ssl => true  
            ssl_certificate_verification => true  
            cacert => '/your_path/elasticsearch-ca.pem'  
        } 
    } 
    ```

# Logstash 與 Filebeat 之間
這一步比較特別，`Logstash`與`Filebeat`需要雙向驗證，也就是兩邊都是`Server`角色 

同時也都要認可對方，所以正常來說需要兩份證書，但由於我們共用不管他， 

一樣拿同一份來用，反正重點是資料傳輸加密，信任等等關係不太重要，都是內部

另外一個重點是`Logstash`與`Filebeat`格式不同，需要提取出`crt`及`key`，來使用 

並且`Logstash`的`key`只接受`pkcs8`，所以還需要轉換 


由於是`Server`端就是拿`http.p12`來取`crt`與`key`喽

1. 取出憑證內的crt及key
    ```sh
    # 取出憑證 
    openssl pkcs12 -info -in http.p12 -out http.crt -nokeys 

    # 取出Key 
    openssl pkcs12 -info -in http.p12 -out http.key -nodes -nocerts 

    # Logstash key 需要轉pkcs8
    openssl pkcs8 -in http.key -to pk8 -n ocrypt -out http.p8 

    # 信任憑證拿之前那份Client用的即可 
    elasticsearch-ca.pem 
    ```
    最終有:

    + http.crt => 證書
    + http.key => 私鑰
    + http.p8 => 私鑰(logstash)
    + elasticsearch-ca.pem(信任憑證)

2. Logstash Pipeline Config => input
    ```ruby
    # vim  conf.d/your.conf 

    input {  
        beats {  
            port => 5044 
            ssl => true  
            ssl_certificate_authorities => ["your_path/elasticsearch-ca.pem"]  
            ssl_certificate => " your_path/http.crt" 
            ssl_key => "your_path/http.p8" 
            ssl_verify_mode => "force_peer"  
        }  
    } 
    ```
3. Filebeat -> output to logstash
    ```yaml
    # vim filebeat.yml 

    output.logstash: 
        hosts: [https://Node1.test.com:5044"] 
        ssl.certificate_authorities: ["/your_path/elasticsearch-ca.pem"] 
        ssl.certificate: "/your_path/http.crt" 
        ssl.key: "/your_path/http.key" 
    ```

# Filebeat 直送 Elasticsearch
`Filebeat`可以直接吃本機的信任憑證，故只需要將`elasticsearch-ca.pem`放好

1. 存放更新本機憑證
    ```sh
    # 丟到ca信任目錄
    mv elasticsearch-ca.pem /etc/pki/ca-trust/source/anchors/

    # 更新
    update-ca-trust
    ```
2. Filebeat => output to elasticsearch
    ```yaml
    # vim filebeat.yml 

    output.elasticsearch: hosts: ["https://Node1.test.com:9200"]  
        protocol: "https" 
    ```