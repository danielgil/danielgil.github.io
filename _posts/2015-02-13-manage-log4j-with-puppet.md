---
layout: post
title: Manage log4j 2.x with Puppet
comments: true
permalink: manage-log4j-with-puppet
tags:
  - log4j
  - puppet
  - devops
  - code
---

In the pasts months, I've found myself in this loop at work:

1. Ship a product and a Puppet module to automate its installation.
2. It's *almost* what the customer wants, except for `<little detail>`.
3. Extend the Puppet module to allow the configuration of that detail, and ship it again.
4. Go back to 2.

The `<little details>` are often ERB templates that need extra customization, and one of the most frequent offenders has been `log4j`, so enough is enough:

{% highlight bash %}
$ puppet module install danielgil-log4j
{% endhighlight %}

This module will use [Augeas](http://augeas.net/) to generate the log4j XML configuration file either from Hiera data or programatically by using the provided Puppet types. Let's see a couple of quick examples.

### From Hiera

You have to configure Hiera to lookup the value of `::log4j::data`. It's a structure of nested hashes. 

The first level, it's simply the name of the configuration file, in this case `/opt/myapp/log4j.xml`. Multiple configuration files can be defined:

{% highlight yaml %}
---
log4j::data:
    '/opt/myapp1/log4j.xml':
        ...
    '/opt/myapp2/log4j.xml':
        ...
    '/someotherapp/config.xml':
        ...
{% endhighlight %}

On the second level there are two keys, `loggers` and `appenders`, each containing yet another nested hash. Each configuration file needs to have a logger and an appenders key.

{% highlight yaml %}
---
log4j::data:
    '/opt/myapp1/log4j.xml':
        loggers:
           ...
        appenders:
           ...
    '/opt/myapp2/log4j.xml':
        loggers:
           ...
        appenders:
           ...
    '/someotherapp/config.xml':
        loggers:
           ...
        appenders:
           ...
{% endhighlight %}

Inside the loggers hash, each key will be the `name` of class you want to log, and it can contain the `level` and the `additivity`.

{% highlight yaml %}
    loggers:
        'com.mycompany.mypkg.myclass':
            level: INFO
            additivity: true
        'org.myorg.pkg.class':
            level: ERROR
{% endhighlight %}

Similarly, each key in the appenders hash will be the `name` of the appender. The possible values inside each appender depend on the `type`, which currently supports *file*, *rollingfile* and *console*.

{% highlight yaml %}
    appenders:
        someappender:
            type          : 'console'
            layout        : '%m%n'
        anotherappender:
            type          : 'rollingfile'
            layout        : '%d{ISO8601} [%t] %-2p %c{1} %m%n'
            policy_startup: false
            policy_size   : '200 Mb'
{% endhighlight %}


### Programatically

This module provides a few types to be used in the Puppet manifests: `configfile`, `logger`, and one type per appender, `appenders::file`, `appenders::rollingfile` and `appenders::console`.

To start, declare the configfile resource:
{% highlight puppet %}
log4j::configfile {'/tmp/config.xml':
  user            => 'root',
  group           => 'root',
  mode            => '0644',
  monitorInterval => '40',
  rootLevel       => 'INFO',
  replace         => false,
  xmllint         => true,
}
{% endhighlight %}

Next, loggers can be added to it. Notice that the `path` in the logger must match the name of the configfile:

{% highlight puppet %}
log4j::logger {'my.class':
  path       => '/tmp/config.xml',
  level      => 'INFO',
  additivity => true,
}
{% endhighlight %}

Finally add the appenders. Once again, `path` must match the configfile:

{% highlight puppet %}
log4j::appenders::console {'stdout':
  path     => '/tmp/config.xml,
  follow   => true,
  target   => 'SYSTEM_ERR',
  ignoreexceptions => true,
  layout   => '%m%n',
}
{% endhighlight %}

### Conclusion
This doesn't solve the real problem, which is that in *Configuration Management Systems*, the line between data and code is often blurry, but at least alleviates the situation of one of the most common cases in my experience, log4j.

In the [project's repo](https://github.com/danielgil/log4j) you can find the code and some more examples.

