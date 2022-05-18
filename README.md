# ELK
> 個人應用紀錄
* 版本：7.x~7.16.2

# 基本介紹
主要以維運架設為主，主要目的為收集Log，非當應用資料庫使用

+ Kibana

    Web服務，提供將資料圖形化，包含檢測設定的中控平台
    
    預設Port => 5601

+ ElasticSearch
    
    資料庫，負責儲存資料

    預設存取Port => 9200

+ Logstash

    負責檔在ElasticSearch前面作為過濾/客製化Log資訊用

    預設Port => 5044

+ Beat系列

    屬於Agent軟體，安裝在客端設備收集客端發送到ElasticSearch或Logstash

    內部已支援大多服務的Log形式模組，也能客製化