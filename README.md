此篇文章記錄使用 FLASK 開發 + APACHE 部屬在 AWS 上時所遇到的問題及解決方法，並使用 cloudflare 的轉址服務

# FLASK 開發
FLASK 開發網路上已經有很多資源，個人主要是參考下面網址進行開發，當然裡面也有些許的 BUG 需要自行排除，本人沒有特別紀錄。\
https://hackmd.io/@shaoeChen/HJiZtEngG/https%3A%2F%2Fhackmd.io%2Fs%2FS1dY8vepQ

## 一、AWS 使用
關於 FLASK + AWS EC2 網路上也有許多例子，雖然 aws 內有 EB 可使用，但因為我有用到 python 中某個套件，需要去改 module 內的 import，所以選用 ec2。如果之後是要使用 apache 做服務的話，建議最少使用 small (RAM >= 2GB) 的機器，因為 apache 很吃資源，當記憶體使用量到 80% 左用，LINUX 系統的 SWAP 會一直使用，導致 CPU 使用率居高不下，最後讓整個 instance 當機需要重開。再把你的 FLASK 丟上 EC2 前，記得先去安全群組，把 port 22 限制成自己的 ip 會比較安全，之後就可以用 ssh 連進你的實例中，並記得開啟 HTTP 與 HTTPS 的所有流量。

## 二、 APACHE2 部署
有關 APACHE2 的設定大致上是根據下面網址做的 https://jqn.medium.com/deploy-a-flask-app-on-aws-ec2-1850ae4b0d41 \
進入後第一步就是先更新套件，否則可能會無法安裝 apache 跟 wsgi。
```
sudo apt-get update
sudo apt-get install apache2
sudo apt-get install libapache2-mod-wsgi-py3
```
裝好之後就可以連入 ec2 實例的公有 ip，不過現在都是預設走 https，但 apache 啟用 https 還需要匯入公私鑰，所以可以先連 http 的網址試試，如果成功就會看到以下圖片 \
![img](https://ubuntucommunity.s3.us-east-2.amazonaws.com/original/2X/7/771159b35c97e429247aac754ad44bf06cc1efa8.png) \
在來就是要啟用 https，在此我們可以使用 openssl 來創建公私鑰，第一步就是先裝 openssl，再來啟用 apache 的 ssl 模組，然後重開 apache。
```
sudo apt-get install openssl
sudo a2enmod ssl
service apache2 restart
```
照著以上步驟後會在 ```/etc/ssl``` 裡面看到許多密鑰對，啟用 apache ssl 模組後在 ```/etc/apache2/sites-available``` 中也會產生 ```default-ssl.conf```，裡面就自動幫你連接到 openssl 的密鑰對，此時就可以使用 https 連到實例的公有 ip 了。當然也可以執行以下指令來產生公私鑰，就會產生 server.key，2048 是使用 rsa 長度不少於 2048 位元加密
```
openssl genrsa -des3 -out server.key 2048
```
在過程中會要求輸入密碼和一些問題，其中密碼一定要牢記，然後就去一些機構做認證。

## 三、部署 flask
確定外部可以連到你的 ec2 實例後，接下來就是要將外部使用者連到你的 flask 網站。首先先安裝 pip 並用此套件來安裝 flask
```
sudo apt-get install python-pip
sudo pip install flask
```
ubuntu 24.04 會建議你不要安裝在 root 底下，另外開一個虛擬環境來控管，這個就取決於每個人，我是選擇裝在 root 下。然後開啟 flask 專案資料夾，並在內部寫個 app.py
```
mkdir flaskapp && cd flaskapp
vi app.py
```
flaskapp 中至少會有 app.py 與一個 flaskapp.wsgi，後者晚點會提到作用。app.py 放以下內容，並先在本地執行測試
```
from flask import Flask
app = Flask(__name__)
@app.route('/')
def hello_world():
    return 'Hello from Flask!'
if __name__ == '__main__':
    app.run()
```
輸入 ```python3 app.py``` 若成功則會看到以下訊息
```
 * Serving Flask app 'app_blog'
 * Debug mode: on
WARNING: This is a development server. Do not use it in a production deployment. Use a production WSGI server instead.
 * Running on http://127.0.0.1:8080
Press CTRL+C to quit
 * Restarting with stat
 * Debugger is active!
```
連上顯示的網址，看到```Hello from Flask!```這段文字就是沒問題，接下來就是利用 wsgi 來將 flask 送上 web。\
flaskapp.wsgi 放以下內容
```
import sys
sys.path.insert(0, '/var/www/html/flaskapp')
from app import app as application
```
接著 sudo vi /etc/apache2/sites-available/flaskapp.conf 並加入以下內容
```
<VirtualHost *:80>
    ServerName {public IP}

    WSGIDaemonProcess flaskapp threads=5
    WSGIScriptAlias / /home/ubuntu/flaskapp/flaskapp.wsgi

    <Directory /home/ubuntu/flaskapp>
        Options Indexes FollowSymLinks
        Require all granted
        Allow from all
    </Directory>
</VirtualHost>

<VirtualHost *:443>
    ServerName {public IP}

    WSGIScriptAlias / /home/ubuntu/flaskapp/flaskapp.wsgi

    <Directory /home/ubuntu/flaskapp>
        Options Indexes FollowSymLinks
        Require all granted
        Allow from all
    </Directory>
</VirtualHost>
```
並建立一個捷徑連結到 /var/www/html/flaskapp 並將該捷徑的使用者改為 www-data，最後在重啟 apache，連入公有 IP 應該就能看到 flask 了。
```
sudo ln -s flaskapp /var/www/html/flaskapp
sudo chown -R www-data:www-data /var/www/html/flaskapp
sudo service apache2 restart
```
如果最後是顯示 Internal server error，可以 ```vi /var/log/apache2/error/log``` 看是什麼問題。如果是 permission denied，則有可能是沒有做到 chown，找不到某文件可能是 ln -s 或是路徑寫錯，可以多做檢查。之後有改 python 內的 code，記得都要 sudo service apache2 restart 才會有效果，如果只是改前段的 html/js/css 之類的則不需要。

## 四、利用 CLOUDFLARE 轉到個人網址
在網路服務中，網址通常是必買的，當然可以全部都走 aws 的服務，不過就我打聽到的價錢是 CLOUDFLARE 比較便宜，而且有免費的 SSL/TSL 證書可以用，由於 CLOUDFLARE 是走 https 的設定，所以記得要開啟 https 的 port 讓他走，加密模式要選完整，並且要修改 flaskapp.conf 中的 ServerName，再把 Public IP 貼過去就會立即生效了。

# 為何不用 ngnix
一開始使用 ngnix 架設服務，結果只能連上一個 app.route，其他都連不上，不知道是哪裡沒設定好，但改用 apache 直接成功連上。不過若哪天 nginx 成功連上其他 app.route 就會改用，畢竟記憶體使用量差太多了。
