---
layout: post
title: Dockerizing a Puppetized App
comments: true
permalink: dockerizing-a-puppetized-app
tags:
  - docker
  - puppet
  - sed
  - tomcat
---

...and butchering the English language with cool new verbs in the process!

### The Problem
In *bulletized* form:

* I have a multi-process app that currently is configured and run with Puppet, gathering configuration data from Hiera and/or facts.
* The application needs host-specific configuration to run.
* I want to *dockerize* it, but a Docker image should not contain host-specific configuration.
* Thus, I cannot just run Puppet in the Dockerfile, as that would *nonportabilize* the Docker image.

Until now I've been using Puppet to **build** the environment and **run** the application. If the application was more agnostic of the environment, it wouldn't be a problem to let Puppet configure it, save it into a Docker image and be able to run it anywhere, but since it's pretty tied to it, this information needs to be gathered at run time to keep the image portable.

It's completely possible that I'm missing something obvious, but achieving portability in this scenarios is proving to be challenging.


### The Solution (until I find a real solution)

The first step is to replace real data with tokens that can be easily found with *sed* in the Hiera backend. For example, if my application needs to know its public DNS name, before I would just pass the value to Hiera, or use the fqdn fact:

{% highlight yaml %}
---
myapp::hostname:  %{fqdn}
myapp::port    :  443
{% endhighlight %}

So here the values are changed by static tokens:
{% highlight yaml %}
---
myapp::hostname:  PUBLICDNS
myapp::port    :  PUBLICPORT
{% endhighlight %}

In the Dockerfile, the RUN command will configure the application using these tokens with *puppet apply*. However, when launching the container, these tokens will be replaced by environment variables by sed. Here's an example Dockerfile:

{% highlight docker %}
FROM base
MAINTAINER Daniel Gil <some.email@example.com>

# Expose ports
EXPOSE 8080

# Configure the application
RUN puppet apply /root/manifests/myapp.pp

# Default container command
CMD sed -i "s/PUBLICDNS/${PUBLICDNS}/"  /etc/myapp/abcd.conf && \ 
    sed -i "s/PUBLICPORT/${PUBLICPORT}/" /etc/myapp/efgh.conf && \ 
    service myapp start && \ 
    tail -f /var/log/myapp.log
{% endhighlight %}

And here's how I would build the image and launch the container, passing the environment variables (or if there were too many, using --env-file):
{% highlight bash %}
$ docker build -t myapp .
$ docker run -d --env PUBLICDNS="myapp.mydomain.com" --env PUBLICPORT="443" --name myapp1 myapp
{% endhighlight %}

Using this technique, I have a generic image that can be run anywhere by passing the host-specific data into it through the command line. It's ugly, but it will have to suffice until I can find a better approach.

### Conclusion
I need to read and learn more about Docker, I have the feeling I'm doing something wrong and that this solution is too convoluted. I could always configure the application at run time with Puppet:

{% highlight docker %}
CMD puppet apply /root/manifests/myapp.pp \ 
    service myapp start && \ 
    tail -f /var/log/myapp.log
{% endhighlight %}

But this defeats the purpose of using Docker, as the configuration includes slow operations like downloading java artifacts with maven, checking out from git, etc.
















