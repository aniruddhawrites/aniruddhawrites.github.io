---
layout: post
title: "Idea on JAMstack, Markdown and a Drupal module, part 3 — Microservices and why it’s essential"
subtitle: "Microservices are the most important part of JAMstack, the A stands for API, is the only feature with which a static site can keep it’s fast, secure, robust ness, having a dynamic behavior."
cover-img: /assets/img/JAMstack-architecture.png
thumbnail-img: /assets/img/JAMstack-architecture.png
share-img: /assets/img/JAMstack-architecture.png
tags: [JAMstack, markdown, Drupal, ideas, migration, platform, headless]
author: Aniruddha Banerjee
---
According to wikipedia, COMPOSER is an application-level package manager for PHP, and we will learn few more features about it. ![according-to-wikipedia-composer-definition](https://bloganiruddha.files.wordpress.com/2018/06/according-to-wikipedia-composer-definition.png)   Download & Setup

*   Windows : Download and run [Composer-Setup.exe](https://getcomposer.org/Composer-Setup.exe)
*   Preferred way:
    *   Know your PHP version open cmd or gitbash and run _php –v_
    *   Prepare a shell script/bat file (composersetup.sh/composersetup.bat)
    *   Content of the shell script executable are:

`php -r "copy('https://getcomposer.org/installer', 'composer-setup.php');"` `php -r "if (hash_file('SHA384', 'composer-setup.php') === '544e09ee996cdf60ece3804abc52` `599c22b1f40f4323403c44d44fdfdd586475ca9813a858088ffbc1f233e9b180f061’)` `{ echo 'Installer verified'; }` `else { echo 'Installer corrupt’;` `unlink('composer-setup.php'); } echo PHP_EOL;"` `<span style="color:, 0000ff;">**php composer-setup.php**</span>` `php -r "unlink('composer-setup.php');"`   Perhaps you noticed the php execution statement is in blue color. Whats in it?   ![i-can-do-more](https://bloganiruddha.files.wordpress.com/2018/06/i-can-do-more.png) So with these options you can setup composer : `php composer-setup.php --install-dir=bin --filename=composerbin --version=1.0.0-alpha8` So how dependencies injected and

*   What are your need
*   How composer resolves them
*   What is dependency injected

Answers are:

* * *

*   Composer is **not **a package manager
*   Composer deals with "packages" or libraries, can manage per-project basis
*   Installing them in a directory (e.g. **vendor**) inside your project (configurable)
*   Composer is a dependency manager. It Injects package dependency via autoload psr-4
*   Composer is built inspired by node's npm and ruby's bundler

**Your Need:**

*   You have a project that depends on a number of libraries.
*   Some of those libraries depend on other libraries.

**Composer** **solution:**

*   Enables you to declare the libraries you depend on.
*   Finds out which versions of which packages can and need to be installed
*   and installs them (meaning it downloads them into your project and inject).

* * *

Now its time to show you an example of Composer.json in detail view: ![composer-dot-json-a-detail-view](https://bloganiruddha.files.wordpress.com/2018/06/composer-dot-json-a-detail-view.png)  

<div id="Fig-3">Source: [Getcomposer.org](https://getcomposer.org/doc/06-config.md#config)</div>

Fig-3 You created a composer.json file for your project, so now its time to install the dependencies [require(for all environment) and require-dev(for debugging environment only)]. The command is: composer install --prefer-source -o ...there are so many arguments you can pass to change the installation procedure and bring stability and so more....I just quoted options from [GetComposer.org](https://getcomposer.org/doc/03-cli.md#install) itself:

### Options

*   **--prefer-source:** There are two ways of downloading a package: `source` and `dist`. For stable versions Composer will use the `dist` by default. The `source` is a version control repository. If `--prefer-source` is enabled, Composer will install from `source` if there is one. This is useful if you want to make a bugfix to a project and get a local git clone of the dependency directly.
*   **--prefer-dist:** Reverse of `--prefer-source`, Composer will install from `dist` if possible. This can speed up installs substantially on build servers and other use cases where you typically do not run updates of the vendors. It is also a way to circumvent problems with git if you do not have a proper setup.
*   **--dry-run:** If you want to run through an installation without actually installing a package, you can use `--dry-run`. This will simulate the installation and show you what would happen.
*   **--dev:** Install packages listed in `require-dev` (this is the default behavior).
*   **--no-dev:** Skip installing packages listed in `require-dev`. The autoloader generation skips the `autoload-dev` rules.
*   **--no-autoloader:** Skips autoloader generation.
*   **--no-scripts:** Skips execution of scripts defined in `composer.json`.
*   **--no-progress:** Removes the progress display that can mess with some terminals or scripts which don't handle backspace characters.
*   **--no-suggest:** Skips suggested packages in the output.
*   **--optimize-autoloader (-o):** Convert PSR-0/4 autoloading to classmap to get a faster autoloader. This is recommended especially for production, but can take a bit of time to run so it is currently not done by default.
*   **--classmap-authoritative (-a):** Autoload classes from the classmap only. Implicitly enables `--optimize-autoloader`.
*   **--apcu-autoloader:** Use APCu to cache found/not-found classes.
*   **--ignore-platform-reqs:** ignore `php`, `hhvm`, `lib-*` and `ext-*` requirements and force the installation even if the local machine does not fulfill these. See also the [`platform`](https://getcomposer.org/doc/06-config.md#platform) config option.

Have you observed the script section in <a id="Fig-3"></a>Fig-3 right. You can run any number of  pre, post and custom scripts with dependency installation. e.g. Symfony 3.4 version does: `"symfony-scripts": [` `        "Incenteev\\ParameterHandler\\ScriptHandler::buildParameters",` `        .......................` `],` `"post-install-cmd": [` `        "@symfony-scripts"` `],` `"post-update-cmd": [` `        "@symfony-scripts"` `]` So it defines a names script "symfony-script" and running at pre and post installation. Interesting dahh!!! Once your installation process is done, composer will create a lock file.

*   composer.lock file have all version specific details of packages, locking the project to those specific versions
*   So, you understand that this is a best practice to commit the composer.lock file, so others working on the project are locked to the same versions of dependencies
*   and If no composer.lock file present, Composer simply resolves all dependencies listed in your composer.json file and downloads the latest version of their files (remember lock helps in caching as well, location is<c:\users\user_name\appdata\roaming\composer></c:\users\user_name\appdata\roaming\composer>

Likewise for my project the lock file look like: ![composer-dot-lock-file](https://bloganiruddha.files.wordpress.com/2018/06/composer-dot-lock-file.png)   Pretty much done..last and not the least. Why we are installing via composer, aren't those packages not having downloadable links today !!!! Indeed, all these dependencies can be manually downloaded . Composer basically looks a repo list : [Packagist](https://packagist.org/). Go and search for "doctrine/doctrine-bundle" and you will see :   ![packagist-the-repo-list](https://bloganiruddha.files.wordpress.com/2018/06/packagist-the-repo-list.png) Go to the details page of the bundle/package..   ![packagist-knows-where-the-bundle-can-be-downloaded](https://bloganiruddha.files.wordpress.com/2018/06/packagist-knows-where-the-bundle-can-be-downloaded.png) Notice the homepage url and its github. So you can download it from github easilty. So again why composer..... Simply said, to add the package, in earlier days, you have to use the package namespace as require. But new things evolved. Now psr-4 autoload can do it for you. You notice the below line in composer.json, I am copying as Symfony 3.4 version provides.. `"autoload": {` `        "psr-4": {` `        "AppBundle\\": "src/AppBundle"` `    },` `   "classmap": [` `        "app/AppKernel.php",` `        "app/AppCache.php"` `    ]` `}` so it gives you the bootstrap and there will be a "autoload.php" inside your vendor directory, it will take care the bootstrapping for you. Walla!!!!!!! stay tuned for the next phase of this topic..TC
