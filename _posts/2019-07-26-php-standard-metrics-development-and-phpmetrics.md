---
layout: post
title: "PHP standard metrics development and PHPMetrics"
subtitle: "PHPMetrics provides tons of metrics, Complexity: Cyclomatic complexity, Myer's
  interval, Relative system complexity, Volume: Vocabulary, Data complexity,
  Lines of code, Readability..., Object Oriented: Lack of cohesion of methods,
  Coupling, Abstraction..., Maintainability: Maintainability index, Halstead's
  metrics, Effort..., And many more ! An excerpt"
cover-img: /assets/img/path.jpg
thumbnail-img: /assets/img/thumb.png
share-img: /assets/img/path.jpg
tags: [modern-web-technology, coding-standard, code-sniff, code-complexicity]
author: Aniruddha Banerjee
---
What is PHPMetrics and how can I use it?

* PHPMetrics provides tons of metrics:
* Complexity: Cyclomatic complexity, Myer's interval, Relative system complexity
* Volume: Vocabulary, Data complexity, Lines of code, Readability...
* Object Oriented: Lack of cohesion of methods, Coupling, Abstraction...
* Maintainability: Maintainability index, Halstead's metrics, Effort...
* And more !

As said by [PHPMEtrics.org](http://www.phpmetrics.org/), its a static analysis tool for PHP

So its is pretty much defined there, few things I want to highlight.

Install as a dev-dependency, your production server should not be loaded with it. So the composer.json entry should be:

```
"require-dev": {
```

```
        "phpmetrics/phpmetrics": "2.3.2",
```

```
        ...
```

```
}
```

Or, you can download the phar from github, and run as:

php /c/bin/phpmetrics.phar --report-html=metrics-report .

Now you have to link /metric-report/index.html so that it can be published to reviewers.

Pretty cool right!! You want a Drupal 8 module...if so check my phpmetricsintegration module, usuage are in project page (Y)
