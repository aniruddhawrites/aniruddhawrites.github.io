---
templateKey: post
title: How to add successive crons via puppet
author: Aniruddha
date: '2018-07-19 10:07:32'
introImage: /assets/img/puppet-manifest-cron.png
tags: '#infrastructure #installation #configurations'
intro_paragraph: Puppet is most popular manifest for Infrastrure Automation. I described a very popular issue, that faced, and how it resolved.
path: "how-to-add-successive-crons-via-puppet"
---
I want to add successive crons via puppet, first one to set as each 10 minutes, and the 2nd one to run in Sunday 7:00PM. The first cron in puppet is working properly, but the 2nd one shows the below error: "Error: Could not retrieve catalog from remote server: Error 400 on SERVER: Invalid relationship: Cron[notifyinactivetargetweekly] { require => File[...notifyinactivetargetweekly.sh] }, because File[...notifyinactivetargetweekly.sh] doesn't seem to be in the catalog Warning: Not using cache on failed catalog Error: Could not retrieve catalog; skipping run" Below are the manifest code.

    cron { 'firstsynccron':
        command => "${phpapp::DocRootDir}/firstcronsync.sh ${scmdemophp::Environment} ${phpapp::DocRootDir}",
        require => File["${phpapp::DocRootDir}/firstcronsync.sh"],
        minute  => '*/10',
        environment=>["COMPOSER_HOME=${phpapp::DocRootDir}",
                        "SYMFONY_ENV=${phpapp::Environment}",
                        "SYMFONY_DEBUG=${phpapp::Debug}",
                        "PATH=.../usr/local/sbin:/usr/local/bin:/sbin/:/bin/:/usr/sbin/:/usr/bin/"
                    ],

    }->
    cron { 'notifyweeklycron':
        command => "${phpapp::DocRootDir}/notifyweeklycron.sh ${phpapp::Environment} ${phpapp::DocRootDir}",
        require => File["${phpapp::DocRootDir}/notifyweeklycron.sh"],
        minute  => '00',
        hour  => '19',
        weekday  => 'Sunday',
        environment=>["COMPOSER_HOME=${phpapp::DocRootDir}",
                        "SYMFONY_ENV=${phpapp::Environment}",
                        "SYMFONY_DEBUG=${phpapp::Debug}",
                        "PATH=.../usr/local/sbin:/usr/local/bin:/sbin/:/bin/:/usr/sbin/:/usr/bin/"
                    ],

    }->

The hieradata consists the below classloading:

    classes:
      - phpapp::mysymfonydeploy

the init.pp of edsconsoledeploy consists:

    class scmdemophp {
        $DocRootDir = "/var/code"

and I checked that the file "/app/code/notifyinactivetargetweekly.sh" exists. UPDATES: I have created the same, but for some reason the cron is not running as per timing:

    file { "${DocRootDir}/notifyweeklycron.sh":
      ensure  => file,
      content => "... whatever's in the file, or use a template/source ...",
    }->
    cron { 'notifyweeklycron':
        command => "${phpapp::DocRootDir}/notifyweeklycron.sh ${phpapp::Environment} ${phpapp::DocRootDir}",
        require => File["${phpapp::DocRootDir}/notifyweeklycron.sh"],
        minute  => '*/15',
        environment=>["COMPOSER_HOME=${phpapp::DocRootDir}",
                        "SYMFONY_ENV=${phpapp::Environment}",
                        "SYMFONY_DEBUG=${phpapp::Debug}",
                        "PATH=.../usr/local/bin:/sbin/:/bin/:/usr/sbin/:/usr/bin/"
                    ],

    }

but its not running after 15 minutes, need help

1.  The puppet log says: File[/var/code/firstsynccron.sh]/mode: mode changed '0664' to '0751'
2.  but it does not showing the same for File[/var/code/notifyweeklycron.sh]/mode: mode changed '0664' to '0751'
3.  frequency changes are reflecting though

I was so confused and searching what to do, I received an answer on a tech forum, and that I want to describe here: What I was missing was using a `require`, `before`, `subscribe` or `notify` parameter, which says that a resource is related to a file or other resource must contain a valid reference. The `require` is requiring a file _resource_ which is defined in your Puppet manifests, and also surprisingly it does not necessarily a file on the server location. So what was there in my code:

    require => File["${phpapp::DocRootDir}/notifyinactivetargetweekly.sh"],

So puppet was expecting, a file resource destined at `/var/code/notifyweeklycron.sh`should exist as per my manifest, and the fix was:

    file { "${phpapp:DocRootDir}/notifyweeklycron.sh":
      ensure  => file,
      content => "... whatever's in the file, or use a template/source ...",
    }

My `require` issue has been resolved.
