# bitbucket repository + drone (redis+postgres) + ssh example

# 在測試主機上產生 ssh key  
```
# 產生key 注意要用RSA
ssh-keygen -t rsa -b 4096 -C "你的Email"
# 名稱輸入(可自訂)
deploy-ssh
# 將公鑰加入到允許清單 (此使用root 建議使用其它帳號)
cat ./deploy-ssh.pub >> /root/.ssh/authorized_keys
```

# 手順
1. 先產生 ssh key 並加入至 authorized_keys
2. 至 Bitbucket → Workspace settings → OAuth consumersOAuth 建立 OAuth 產生 Key 及 Secret ，並填回 docker-compose.yml 內
```
Name: drone CI
Callback URL: http://ci.abc.com
This is a private consumer: 打勾

Permissions:
    Account: Email,Read
    Issue: Read
    Workspace: Read
    Webhooks: Read and write
    Repositories: Read, Write
    Pull requests: Read
```
3. 在主機上運行 docker-compose
```
# 請參照檔案 drone/docker-compose.yml

# 啟動 docker-compose
cd drone
docker-compose up -d

# 停止
docker-compose stop

# 刪除重建(drone內資料都會刪除請注意，亦可另外建 volume 存放資料)
docker-compose down

# 檢查服務是否有啟動
netstat -tlnp|grep 8080
```
4. 設定 nginx 反向代理
```
server {
    listen 80;
    server_name ci.abc.com;

    location / {
        proxy_set_header X-Forwarded-For $remote_addr;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_set_header Host $http_host;

        proxy_pass http://127.0.0.1:8080;
        proxy_redirect off;
        proxy_http_version 1.1;
        proxy_buffering off;

        chunked_transfer_encoding off;
    }
}
```
5. 登入 drone
```
開啟瀏覽器 http://ci.abc.com ，此時會需要登入授權
```
6. 專案啟動 webhooks
```
# 啟用 webhook
登入 drone 後到要自動部署的專案點擊 ACTIVATE REPOSITORY
則會自動將 bitbucket 的 webhook 設定

# 開啟授權
Settings 頁籤內將 Trusted 打勾

# 設定相關 Secrets 
 (大小寫要與 .drone.yml 內的 from_secret 一致)   
  
PROJECT_PATH 設定測試機上的專案路徑，ex: /home/www
SSH_HOST 設定測試機主機位置，ex: ci.abc.com 
SSH_PRIVATE_KEY 將先前產生的 deploy-ssh (private key) 內容貼至此
SSH_USERNAME SSH登入帳號，ex: root
```
7. 在專案根目錄建立 .drone.yml
```
請參照檔案 .drone.yml
```
8. git push 之後應該就能在 drone 的 builds 頁籤內看到結果了