---
layout: post
title: "SimpleSAML Installation in Windows Apache"
subtitle: "Setting up SimpleSAMLphp in windows Apache/Xampp kills my one day once ago. With a very limited documentation, what I able to achieve, I explained in this post. Let me know if you liked/disliked it."
cover-img: /assets/img/path.jpg
thumbnail-img: /assets/img/thumb.png
share-img: /assets/img/path.jpg
tags: [apache, configurations, SAML, single-signon]
author: Aniruddha Banerjee
---
Say you want to install SimpleSAMLphp in windows Apache/Xampp. Let me tell you what I did so far:

  

 1. I have downloaded latest stable version from
    [https://simplesamlphp.org/download](https://simplesamlphp.org/download)
    and placed the unzipped file in Apache folder, i.e.
    C:\Apache24\simplesamlphp directory contains composer.json. I have
    downloaded dependencies as well. 
    
 2. Now when I am going to setup the vhost as shows in the site 6. Configuring Apache section as

```
<VirtualHost *:80>
        ServerName localhost
        DocumentRoot C:/Apache24/htdocs


<VirtualHost *:80>
        ServerName service.example.com
        DocumentRoot C:/Apache24/service.example.com

        Alias /simplesaml C:/Apache24/simplesamlphp/www
```

 3. Changed the config file  

Now I faced the problem: I am unable to open the Alias in browser. And running httpd.exe in browser shows error about the example.com does not exist.  

Later I found that I was doing some mistake... I am going to add one vhost, rather in Windows Xampp, there is one file /xampp/apache/conf/extra/httpd-xampp.conf. Need to add one Alias below /phpmyadmin statement:  

```
Alias /simplesaml "C:/xampp/simplesamlphp/www"
<Directory "C:/xampp/simplesamlphp/www">
        AllowOverride AuthConfig
        Require all granted
</Directory>
```
Now restart server and hit [http://localhost/simplesaml/](http://localhost/simplesaml/) and all you go...Enjoy you SAML!!!!!!!!!!!!!!!!
