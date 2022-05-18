# 多行Log Filter
預設在Filebeat中他會一行行讀取，送出資料每一行都是一個doc

但若有一筆Log是多行的，那就需要把屬於同一筆的Log行數都合併在一起

# 範例
* 說明

    當觸發Exception時會拋出Exception訊息

    第一行是ExceptionName，二,三行是其細節錯誤程式碼行數說明

    但他們算同一時間同一筆輸出，所以必須要bind在一起

    而又剛好中間穿插Info一般訊息
    ```
    10:47:54,668 INFO  [stdout] (http-/192.168.1.1:80) Message1
    10:47:54,668 ERROR [stderr] (http-/192.168.1.1:80) java.Event1Exception: ERROR
    10:47:54,668 ERROR [stderr] (http-/192.168.1.1:80)         at ???(java:106)
    10:47:54,668 ERROR [stderr] (http-/192.168.1.1:80)         at ???(java:108)
    10:47:54,668 INFO  [stdout] (http-/192.168.1.1:80) Message2
    10:47:54,675 ERROR [stderr] (http-/192.168.1.1:80) java.Event2Exception: ERROR
    10:47:54,675 ERROR [stderr] (http-/192.168.1.1:80)         at ???(java:32)
    10:47:54,675 ERROR [stderr] (http-/192.168.1.1:80)         at ???(java:77)
    ```

* 解決方式
    
    ```yml
    # vim /etc/filebeat.yml

    filebeat.inputs:
    - type: log
        enabled: true
        paths:
            - /log_path/log*
        
        # 利用multiline專門來處理多行log
        # 正規表達式，當遇到符合時就收集該行直到在次觸發條件才算終止
        # 因此遇到每個Exception開頭就觸發直到遇到下一個Exception or stdout/INFO一般輸出結束
        # 這樣就會包攬全部的Exception訊息合併一筆
        multiline.type: pattern
        multiline.pattern: 'Exception|stdout'
        multiline.negate: true
        multiline.match: after

    ```

# 參考
> [官方文章](https://www.elastic.co/guide/en/beats/filebeat/current/regexp-support.html)

> [相關文章](https://blog.cti.app/archives/6411)