此篇文章記錄使用 FLASK 開發 + APACHE 部屬在 AWS 上時所遇到的問題及解決方法，並使用 cloudflare 的轉址服務

# FLASK 開發
FLASK 開發網路上已經有很多資源，個人主要是參考下面網址進行開發，當然裡面也有些許的 BUG 需要自行排除，本人沒有特別紀錄。\
https://hackmd.io/@shaoeChen/HJiZtEngG/https%3A%2F%2Fhackmd.io%2Fs%2FS1dY8vepQ

# AWS 使用
關於 FLASK + AWS EC2 網路上也有許多例子，雖然 aws 內有 EB 可使用，但因為我有用到 python 中某個套件，需要去改 module 內的 import，所以選用 ec2。如果之後是要使用 apache 做服務的話，建議最少使用 small (RAM >= 2GB) 的機器，因為 apache 很吃資源，當記憶體使用量到 80% 左用，LINUX 系統的 SWAP 會一直使用，導致 CPU 使用率居高不下，最後讓整個 instance 當機需要重開。\
再把你的 FLASK 丟上 EC2 前，記得先去安全群組，把 port 22 限制成自己的 ip 會比較安全。

# APACHE2 部署
有關 APACHE2 的設定大致上是根據下面網址做的 \
https://jqn.medium.com/deploy-a-flask-app-on-aws-ec2-1850ae4b0d41 
在 APACHE2 中的 config 是在 /etc/apache2/apache2.conf 中，網路上 /etc/httpd/conf/httpd.conf 應該是先前的版本。接下來就看你是要走 http(port 80) 還是 https(port 443)，以現在來說通常是走 https，而上面的設定則是走 http，所以把上面網址中編輯 /etc/apache2/sites-enabled/000-default.conf 的設定放到 /etc/apache2/apache2.conf 中就可以了。如果有設定成功，那麼進入 https://{your ip}/index.html 應該會看到 apache 頁面顯示 It works。\
之後有改 python 內的 code，記得都要 sudo service apache2 restart 才會有效果，如果只是改前段的 html/js/css 之類的則不需要。

# CLOUDFLARE DNS 設定
在網路服務中，網域通常是必買的，當然可以全部都走 aws 的服務，不過就我打聽到的價錢是 CLOUDFLARE 比較便宜，而且有免費的 SSL/TSL 證書可以用，由於 CLOUDFLARE 是走 https 的設定，所以記得要開啟 https 的 port 讓他走，且加密模式要選完整。

# 為何不用 ngnix
一開始使用 ngnix 架設服務，結果只能連上一個 app.route，其他都連不上，不知道是哪裡沒設定好，但改用 apache 直接成功連上。不過若哪天 nginx 成功連上其他 app.route 就會改用，畢竟記憶體使用量差太多了。
