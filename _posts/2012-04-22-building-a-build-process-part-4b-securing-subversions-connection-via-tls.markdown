---
layout: post
title: 'Building a Build Process: Securing Subversion''s Connection via TLS'
date: 2012-04-22 01:01:00.000000000 -04:00
comments: true
series: building-a-build-process
references: 
 - title: http://wiki.centos.org/HowTos/Https
   url: http://wiki.centos.org/HowTos/Https
 - title: http://dev.antoinesolutions.com/subversion/how-to-configure-subversion-on-centos-redhat-linux
   url: http://dev.antoinesolutions.com/subversion/how-to-configure-subversion-on-centos-redhat-linux
 - title: http://www.electrictoolbox.com/changing-hostname-centos/
   url: http://www.electrictoolbox.com/changing-hostname-centos/
   
---

Alright, I just couldn’t take the guilt. I can’t abide Subversion and Apache being set up without defaulting to https and a TLS encrypted connection. We’ll remedy this below.

###Install The Required Packages
On the CentOS server, login as root and type the following in the terminal to install packages:

{% highlight sh %}
    yum install mod_ssl openssl
{% endhighlight %}
    
These packages may already be installed; don’t worry if yum tells you that it has nothing to do.

###Use OpenSSL to Generate a Self-Signed Certificate
Navigate to the certificate store by heading to:

{% highlight sh %}
cd /etc/pki/ca
{% endhighlight %}

Next, we’ll need to create a strong private key (2048-bit encryption):

{% highlight sh %}
openssl genrsa -out ca.key 2048
{% endhighlight %}
    
Then, we generate a CSR (Certificate Signing Request):

{% highlight sh %}
openssl req -new -key ca.key -out ca.csr
{% endhighlight %}
 
You’ll then have to enter the following fields:

* 2-digit country code (e.g. `US`)  
* State or Province full name (e.g. `Maryland`)  
* Locality / City (e.g. `Baltimore`)  
* Organization Name (e.g. `Microsoft` or `Sean Killeen`)  
* Org Unit Name / Section (e.g. `Subversion Test`)  
* Common Name (your new hostname, eg `scm.seankilleen`)  
* Email Address (e.g. `SeanKilleen@gmail.com`)  
* A challenge password  
* An optional company name

The next step is to use the CSR to generate the actual certificate:

{% highlight sh %}
openssl x509 –req –days 365 –in ca.csr –signkey ca.key –out ca.crt
{% endhighlight %}

Then, copy the certificate files into the TLS certs location that we’ll use later:

{% highlight sh %}
cp ca.crt /etc/pki/tls/certs
cp ca.key /etc/pki/tls/private/ca.key
cp ca.cr /etc/pli/tls/private/ca.csr
{% endhighlight %}

###Modify Apache’s Configuration to offer SSL
Open the file for editing:

{% highlight sh %}
gedit +/SSLCertificateFile /etc/httpd/conf.d/ssl.conf
{% endhighlight %}

Find the line referencing “SSLCertificateFile” and change it to the location of your keyfile. It should look like this when you’re done:

{% highlight apache %}
SSLCertificateFile /etc/pki/tls/certs/ca.crt
{% endhighlight %}

A few lines after that, you’ll edit the SSLCertificateKeyFile:

{% highlight apache %}
SSLCertificateKeyFile /etc/pki/tls/private/ca.key
{% endhighlight %}

Save the file and close it.

Next, restart apache – from the console:

{% highlight sh %}
/etc/init.d/httpd restart
{% endhighlight %}

####Redirect Non-SSL connections to the SSL Connection
To do this, we need to edit the apache configuration file. 

{% highlight sh %}
gedit /etc/httpd/conf/httpd.conf
{% endhighlight %}

Towards the bottom of the file, the `<VirtualHosts>` configuration can be found.
Paste the following lines at the bottom of the file:

{% highlight apache %}
NameVirtualHost *:80
NameVirtualHost *:443

<VirtualHost *:80>
        <Directory /var/www>
            AllowOverride All
        </Directory>
        DocumentRoot /var/www
        ServerName localhost
</VirtualHost>
<VirtualHost *:443>
        SSLEngine on
        SSLCertificateFile /etc/pki/tls/certs/ca.crt
        SSLCertificateKeyFile /etc/pki/tls/private/ca.key
        <Directory /var/www>
            AllowOverride All
        </Directory>
        DocumentRoot /var/www
        ServerName localhost
</VirtualHost>
{% endhighlight %}

Now point your browser to *https*://[your ip or host name] and you’ll see that the site loads under an http connection. *NOTE:* you may get an error about the certificate, but this is because it does not come from a CA and thus is not “trusted” by your computer. The encryption is still TLS 1.0 256-bit encryption.

###Enforce SSL/TLS Only (Redirect http to https)
I put this in a separate section because I wanted the additions to be compartmentalized.

To use mod_rewrite to redirect any http requests to https, change the VirtualHost *:80 to the following:

{% highlight apache %}
<VirtualHost *:80>
    RewriteEngine On
    RewriteCond %{HTTPS} off
    RewriteRule (.*) https://%{HTTP_HOST}%{REQUEST_URI}
    <Directory /var/www>
        AllowOverride All
        </Directory>
        DocumentRoot /var/www
        ServerName localhost
</VirtualHost>
{% endhighlight %}

###Configure the Firewall to Allow Port 443 Connections
On the console:

{% highlight sh %}
iptables –A INPUT –p tcp –dport 443 –j ACCEPT
/sbin/service iptables save
iptables –L –v
{% endhighlight %}

Now, try heading to `http:[your ip or hostname]/svn`, and watch it redirect to https. 

Success! 