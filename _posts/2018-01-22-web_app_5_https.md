---
layout: post
title: Building a web app part 5 - Running Apache on HTTPS
category: Programming
tags: ['web development', 'reporting app']
external_links:
    - title: Apache
      link: 'http://httpd.apache.org'
      description: Official documentation for Apache
    - title: Web standard rfc2616
      link: 'https://www.w3.org/Protocols/rfc2616/rfc2616-sec10.html#sec10.3'
      description: Specifications for request redirection
    - title: Guide to setting up self-signed certificates
      link: 'https://www.linux.com/learn/creating-self-signed-ssl-certificates-apache-linux'
---

When I first set up Edinburgh Genomics'
[reporting web app](/programming/2016/07/15/web_app_0_overview.html), it was on
HTTP without SSL encryption. This was not too much of a concern to start with
since our servers were on a closed network, but later as a result of a new
feature being added, it became necessary to switch the app over to use a secure
connection. This meant setting up SSL certificates in order to get the app's
HTTP server to use HTTPS.

## SSL certificates
An SSL certificate is a file that is cryptographically 'signed' by a trusted
party (either an individual or a CA - certificate authority), to enable secure
and authenticated communication between the the client (e.g. a web browser) and
the server. The process of setting up HTTPS simply consists of obtaining SSL
certificates and pointing the web server to them (in the case of Apache, via
`mod_ssl`). This post will describe how I did this on Edinburgh Genomics'
reporting app.

## Self-signed certificates
For testing or personal applications it's perfectly possible to use a
self-signed certificate (after all, the most trustworthy party that can sign
your certificate is yourself). Firstly we need an RSA private key. You might
already have one, e.g. in `~/.ssh`, but you can generate a dedicated one with:

    # openssl genrsa -out rsa_private.key 2048

Once we have a private key, we can then generate a CSR (certificate signing
request):

    # openssl req -new -key rsa_private.key > server_name.csr

This enters an interactive prompt, asking you for metadata to be attached to
the certificate, including your organisation name and location. This information
appears in the resulting CSR file and is used to generate and sign a
certificate:

    # openssl x509 -in server_name.csr -out server_name.cert -req -signkey rsa_private.key -days 100

A certificate is only valid for a finite amount of time, with the default being
30 days. You'll probably want it for longer than that, so use the `-days` option
to specify how long you want the certificate to be valid for. Once you've got
this certificate, you can point your web server to it as below.

## CA-signed certificates
While self-signed certificates are fine for testing and personal applications,
there is an assumption that you trust the person who signed it. Web clients do
not inherently know who to trust, so if you visit a website with a self-signed
certificate, the web browser will complain and programmatic interfaces might not
work at all. This is because web clients only trust a finite list of known CAs -
organisations that are trusted by the worldwide web community to issue signed
certificates. It is possible to install your self-signed certificate into the
client, but that's an extra stage of setup that won't look very professional to
others visiting your website.

The solution to this is to get a certificate signed by a CA. If your
organisation has a process for this already, this is a fairly painless task.
Firstly you need a private key and CSR, generated the same way as for
self-signed:

    # openssl genrsa -out rsa_private.key 2048
    # openssl req -new -key rsa_private.key > server_name.csr

Then instead of generating a certificate yourself, you send the CSR file to your
organisation to be sent for signing.

Once this has happened, your organisation should hand back to you one or more
certificate files. You should have an actual certificate for your server,
potentially a copy of the CA's root certificate (this is public), and an
intermediate certificate linking your server's certificate to the CA's root one.
These can now be set up on your server.

## Adding certificates to Apache
Firstly having got the certificates and associated files onto the server (as
root, and with sensible file permissions!), all that really needs done is to set
up `mod_ssl`. If not already installed:

    # yum install mod_ssl  # or apt-get, or whatever

Then update your web app's Apache config:

    <VirtualHost *:80>  # change this to '*:443'
        ServerName <server_name>

        # add these 4 lines
        SSLEngine on
        SSLCertificateFile "path/to/ssl_certificate.crt"
        SSLCertificateChainFile "path/to/intermediate_certificate.crt"
        SSLCertificateKeyFile "path/to/ssl_key.key"

        # rest of the config...
    </VirtualHost>

Here, we reconfigure the VirtualHost from port 80 (the default for HTTP) to 443
(the default for HTTPS), and add some fairly self-explanatory SSL parameters.
That's pretty much all that's required, however it is possible to go one
further.

## HTTPS redirection

    <VirtualHost *:80>
        ServerName <server_name>  # this can be the same as above
        Redirect / https://<server_url>
    </VirtualHost>

Here we've added another VirtualHost on port 80 which we used to be listening
on, and used Redirect from `mod_alias`, which here simply redirects any
`http://` requests to `https://` on the same server. This will ensure HTTPS is
always used without requiring the user to use `https://` in their URLs. There
is, however, one caveat in that this only works for GET requests.

I discovered this the hard way when I noticed that POST operations via
[Python requests](http://docs.python-requests.org) were returning `200 OK` when
they should have returned `201 Created`. Looking at the response to one of
these, I saw that it had been returned as a GET! It turned out that this is part
of the rfc2616 HTTP specification, which states that any request other than GET
or HEAD that receives a `301 Redirect` (which is how `mod_alias` redirection
works) may not proceed to the new URL. Python requests was then converting this
into a plain GET request (no query parameters) on the same endpoint. Of course
you don't want this kind of behaviour in production, in which case it's
important to use `https://` in the base URL and not rely on redirection. PATCH
requests also failed in Python requests, but at least returned a more sensible
`400 Bad Request`.

Overall setting up HTTPS, despite requiring certificates to be requested from a
third party, was very straightforward on Apache, with the only caveat being
redirection with non-GET requests. I'll definitely be doing this more readily in
the future.
