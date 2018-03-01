---
title: Ubuntu16下配置自己的web服务器（nginx+php-fpm）
categories: 配置环境
tags:
    - 配置环境
    - Web
---

前提工作：先在 www.freenom.com 上申请一个免费的域名，域名与自己的服务器ip绑定

----------------------------------------------
## 搭建nginx服务器(添加php支持)
1. 安装nginx与php-fpm
    ```
    > sudo apt install nginx
    > sudo apt install php-fpm
    ```
2. 为nginx添加站点
    - 在`/etc/nginx/conf.d`目录下新建一个站点的配置文件，假设申请的域名为`abc.tk`，新建`abc.tk.conf`并添加如下内容
    ```
    server {
            listen 80;  # 80是网站默认访问端口
            server_name www.abc.tk abc.tk;
            root /usr/share/nginx/abc.tk;  # 此处为你想设置的文档根目录
            index index.html;

            location / {
            }
    }
    ```
    - 保存退出，在`/etc/nginx/conf.d`目录下用`nginx -t`命令检查文件时候有误
    - 创建web站点文档根目录，注意与前面的配置文件中的root保持一致
3. 为nginx添加php支持
    - 配置` /etc/nginx/sites-available/default`文件
    ```
    location ~ \.php$ {
        include snippets/fastcgi-php.conf;
        #~ # With php7.0-cgi alone:
        #~ fastcgi_pass 127.0.0.1:9000;  #这一行是配置单纯php
        #~ # With php7.0-fpm:
        fastcgi_pass unix:/run/php/php7.0-fpm.sock; #这一行是配置socket，与上边冲突
    }
    ```
    - 配置`/etc/php/7.0/fpm/php-fpm-conf`，在末尾添加
    ```
    listen = /run/php/php7.0-fpm.sock # 与上边配置的socket保持一致
    ```
    - 再修改`/etc/nginx/conf.d/abc.tk.conf`的内容
    ```
    server {
        listen 80;
        server_name www.abc.tk abc.tk;
        root /usr/share/nginx/abc.tk;  #此处为你想设置的文档根目录
        index index.html index.php;  #添加php索引支持

        location / {
        }

        # php-fpm  (新增)
        location ~\.php$ {
                fastcgi_pass unix:/run/php/php7.0-fpm.sock;
                fastcgi_index index.php;
                fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
                fastcgi_param PATH_INFO $fastcgi_script_name;
                include /etc/nginx/fastcgi_params;
        }
    }
    - 修改完后记得`nginx -t`检查一下是否有误
4. 启动并测试
    ```
    sudo service php7.0-fpm start
    sudo service nginx start
    ```
    - 在之前建立的站点根目录中新建测试文件`info.php`
    ```
    <?php
        phpinfo();
    ?>
    ```
    - 最后在客户机访问 abc.tk/info.php测试是否成功，测试结果
    ![测试结果](https://pic1.zhimg.com/80/v2-b5c1c0f2fb44f8f6b7bfb75de428d14d_hd.jpg)
