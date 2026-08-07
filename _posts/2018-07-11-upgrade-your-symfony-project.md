---
layout: post
title: "Upgrade your Symfony project from 2.x to 3.4"
subtitle: "Upgrade your Symfony project from 2.x to 3.4"
cover-img: /assets/img/path.jpg
thumbnail-img: /assets/img/thumb.png
share-img: /assets/img/path.jpg
tags: [misc]
author: Aniruddha Banerjee
---

This article may seems to you extra, as the topic has been covered in many places. There are many other and preferred ways to do this. The reason why I chose the topic, is to show what are the difficulties I have faced while migrating one project. So...hang tight: From Symfony 3.x, the symfony installaer came into light. So to install a Symfony 3.4 version you can opt for any 02 paths: 1\. Using installer: Grab Symfony installer

> https://github.com/symfony/symfony-installer c:\bin\> php -r "file_put_contents('symfony', file_get_contents('https://symfony.com/installer'));" c:\bin\> (echo @ECHO OFF & echo php "%~dp0symfony" %*) > symfony.bat

Install Symfony 3.4 cd to htdocs(I used xampp) and run

> symfony new your_project_name 3.4

![Symfony3.4-via-installer-2](https://bloganiruddha.files.wordpress.com/2018/07/symfony3-4-via-installer-2.png) ![Symfony3.4-via-installer](https://bloganiruddha.files.wordpress.com/2018/07/symfony3-4-via-installer.png) 2\. Using composer:

> composer create-project symfony/framework-standard-edition your_project_name/ "3.4.*"

![Symfony3.4-via-composer.png](https://bloganiruddha.files.wordpress.com/2018/07/symfony3-4-via-composer.png) ![Symfony3.4-via-composer-2](https://bloganiruddha.files.wordpress.com/2018/07/symfony3-4-via-composer-2.png) Now once to change directory to your_project_name, first thing you have to do is to create bundle. You might have installed custom bundle in your Symfony 2.x phase:

> php bin/console generate:bundle --namespace=YourNameSpace/YourBundle

This will ask several inputs, only value to provide is `configuration format` which should be yml (you may need to try twice) ![Symfony3.4-generate-bumdle](https://bloganiruddha.files.wordpress.com/2018/07/symfony3-4-generate-bumdle.png) Now you will face and autoload issue: It will not able to change autoload and the message below will be displayed: The command was not able to configure everything automatically. ![Symfony3.4-aotuload-issue-generate-bumdle](https://bloganiruddha.files.wordpress.com/2018/07/symfony3-4-aotuload-issue-generate-bumdle.png) Now open composer.json and locate the autoload section: ![Symfony3.4-aotuload-issue-composer](https://bloganiruddha.files.wordpress.com/2018/07/symfony3-4-aotuload-issue-composer.png) Change it to "psr-4": { "": "src/" }, Which will looks like: ![Symfony3.4-aotuload-issue-resolved-composer](https://bloganiruddha.files.wordpress.com/2018/07/symfony3-4-aotuload-issue-resolved-composer.png) Now run composer update, see what happened in following figure: ![Symfony3.4-resolve-aotuload-issue-composer-update](https://bloganiruddha.files.wordpress.com/2018/07/symfony3-4-resolve-aotuload-issue-composer-update.png) Acquire dependent bundles Now we must acquire all dependent bundles, need to run

> composer require sonata-project/doctrine-orm-admin-bundle

and so on..... While configure, you will notice many deprecation issue. It depends on bundle configurations. For an example: To remove deprecation, found this https://stackoverflow.com/questions/47698006/symfony-3-4-refreshing-a-deauthenticated-user-is-deprecated, security.yml should looks like:

> main: logout_on_user_change: true anonymous: ~

Import & configure DB Now it’s time to import database: Run

> php bin/console doctrine:database:create --connection=default

This will create database, now import the existing database into newly created one Now copy doctrine folder and paste into src/yourbundle/Resources/config folder run

> php bin/console doctrine:generate:entities YourNameSpace/YourBundle php bin/console doctrine:schema:update –force

Database is configured, now setup vhost and try in browser (add app_dev.php in .htaccess and comment line #13-19 in app_dev.php) Now copy all folders from bundle, except DependencyInjection, Entity, Resources etc. Move views and public assets Copy src/Bundle/Resources/public directory to same location in Symfony 3.4 and run below command

> php bin/console assets:install web php bin/console cache:clear --env=dev

Copy existing src/Bundle/Resources/views files to app/Resources/views Remove all YourNameSpaceYourBundle::” bundle scope from twigs as all templates moved to app/Resources from src/Bundle namespace That's all as of now. Stay tuned !!!
