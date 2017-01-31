+++
title = "Properly Configuring Ngnix for HTTPS"
date = "2015-06-07 10:50:32"
description = "Configure Nginx while avoiding common pitfalls and vulnerabilities"
draft = false
+++

While setting up www.mirango.io, I chose to adopt HTTPS in order to ensure privacy of readers.  Here's a short guide I put together while setting up my server.

## HTTP to HTTPS redirects

In order to enforce HTTPS, it's necessary to redirect all HTTP traffic (port 80) to HTTPS (port 443).  On Nginx servers, this is accomplished within the server block:

```Nginx
server {
       listen         80;
       server_name    www.mirango.com;
       # 301 Permanent Redirect
       return         301 https://www.mirango.io$request_uri;
    }
```

When users visit `http://www.mirango.com:80` they receive a `301 Permanent Redirect` to `https://www.mirango.io:443`.  301 redirects are recommended by both Nginx and [Google][301-google] to redirect traffic from any non-preferred domain.  Similar code is used to specify the canonical domain for mirango.io (www over non-www).

## OCSP Stapling

[OCSP][OCSP], Online Certificate Status Protocol, is a protocol used to check revocation status of certificates - specifically designed as an alternative to certificate revocation lists.  OCSP responders are generally run by certificate authorities, but are often slow and prone to failures.

Enter [OCSP stapling][OCSP-Stapling].  Also known as TLS Certificate Status Request extension,  OCSP stapling allows web servers to cache the OCSP record clients normally receive from an OCSP responder, saving network resources.  The cached record is sent during the TLS handshake, but only if the client requests it during the extended ClientHello (status_request).

```Nginx
ssl_stapling on;
ssl_stapling_verify on;
# make sure full_chain.pem contains all certificates
ssl_trusted_certificate /etc/nginx/ssl/full_chain.pem;
```

## HTTP Strict Transport Security

With the threat of [downgrade attacks][downgrade] on TLS (transparent conversion of HTTPS to HTTP in a MitM style attack), it is recommended to implement HTTP Strict Transport Securit (HSTS). HSTS enables a website to declare it is only accessible through secure connections.

To enable HSTS, simply add the following line to the server block:

```Nginx
# enable HSTS for one year on *.mirango.io
add_header Strict-Transport-Security "max-age=31536000; includeSubDomains";
```

## SSL Protocols and POODLE

The threat from [POODLE][poodle], Padding Oracle On Downgraded Legacy Encryption, has necessitated the retirement of SSLv3 support (TLSv1 actually replaced SSLv3 15 years ago!).  The vulnerability exploited by POODLE allows a MitM attacker to decrypt secure HTTP cookies using a padding oracle attack against CBC-mode (cipher block chaining) ciphers.

POODLE works since SSLv3 decrypts before verifying the MAC (used to detect tampering) and ignores the contents of padding bytes, only looking for the last byte containing the length of the padding.  For a more detailed explanation of the vulnerability that allows POODLE to work, [click here][beast-attack].

Securing a web server against this style attack is as simple as removing SSLv3 support.

```Nginx
# use TLSv1+
ssl_protocols           TLSv1 TLSv1.1 TLSv1.2;
```

## Public Key Pinning Extension for HTTP (HPKP)

HPKP, Public Key Pinning Extension, is a HTTP header that instructs clients to remember the server's cryptographic identity.  When a client visits the server again, it expects a certificate containing the public key stored from the original visit.  HPKP reduces the ability to [issue forged certificates][mitm-attack] for a website.  It should be noted that there is an inherent weakness to key pinning. Since key pinning is a trust on first use security mechanism a client cannot detect a MitM attack (using a forged certificate) when visiting a site for the first time.

HPKP requires the base64-encode of SHA256 hash of the public key, certificate signing request, or the certificate.  [Mozilla provides a short guide][base64]:

```bash
#Given the public key my-key-file.key:
openssl rsa -in my-key-file.key -outform der -pubout | openssl dgst -sha256 -binary | openssl enc -base64

#Given the CSR my-signing-request.csr:
openssl req -in my-signing-request.csr -pubkey -noout | openssl rsa -pubin -outform der | openssl dgst -sha256 -binary | openssl enc -base64

#Or given the certificate my-certificate.crt
openssl x509 -in my-certificate.crt -pubkey -noout | openssl rsa -pubin -outform der | openssl dgst -sha256 -binary | openssl enc -base64
```

```Nginx
add_header Public-Key-Pins 'pin-sha256="BASE64-KEY-INFORMATION"; pin-sha256="BASE64-KEY-INFORMATION"; max-age=5184000; includeSubDomains';
```

## Logjam Vulnerability / FREAK

A recent [paper][logjam-paper] published by INRIA, Microsoft Research, University of Pennsylvania, Johns Hopkins, and University of Michigan exposes a flaw in TLS.  Logjam is similar to FREAK but, rther than an implementation vulnerability like FREAK, logjam exploits a vulnerability in the TLS protocol itself.

The vulnerability relies on old export ciphers that were developed during the 1990's when the United States banned the export of "strong ciphers" ([Arms Export Control Act][aeca-wiki]).  These export ciphers were specifically designed to be weak and today can be broken with home computers.

This leads into the vulnerability.

During a man in the middle attack, an attacker can intercept a client connection and replace accepted ciphers with DHE_EXPORT.  The server will then pick 512-bit parameters, finish remaining computations, and sign the parameters.

The original client, unaware of the MitM attack, believes the server picked a DHE (not DHE_EXPORT) key exchange.  At this point, since the connection uses insecure ciphers, the attacker can easily recover the connection key.  Over 8% of top one million HTTPS websites are vulnerable.

It's now believed that in addition to export Diffie-Hellman, non-export Diffie-Hellman is possibly vulnerable to [nation-state surveillance][NSA-spiegel].  Researchers speculate that the NSA has completed the pre computation (estimated at 48 million core years) of a few common 1024-bit parameters (non unique prime numbers that are shared by many servers) and can now easily break new discrete logarithms.  If the cryptography algorithm is broken, that means that forward security is no longer maintained.

[Mozilla][mozilla-SSL] and [others][logjam-SSL] recommend disabling any export ciphers and using a large prime, preferably unique, for Diffie-Hellman key exchanges.

```bash
# generate unique prime for Diffie Hellman Exchanges
openssl dhparam -out dhparam.pem 4096
```

```Nginx
# prefer Eliptic Curve Diffie Hellman Exchanges
ssl_ciphers 'ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES256-GCM-SHA384:ECDHE-ECDSA-AES256-GCM-SHA384:DHE-RSA-AES128-GCM-SHA256:DHE-DSS-AES128-GCM-SHA256:kEDH+AESGCM:ECDHE-RSA-AES128-SHA256:ECDHE-ECDSA-AES128-SHA256:ECDHE-RSA-AES128-SHA:ECDHE-ECDSA-AES128-SHA:ECDHE-RSA-AES256-SHA384:ECDHE-ECDSA-AES256-SHA384:ECDHE-RSA-AES256-SHA:ECDHE-ECDSA-AES256-SHA:DHE-RSA-AES128-SHA256:DHE-RSA-AES128-SHA:DHE-DSS-AES128-SHA256:DHE-RSA-AES256-SHA256:DHE-DSS-AES256-SHA:DHE-RSA-AES256-SHA:AES128-GCM-SHA256:AES256-GCM-SHA384:AES128-SHA256:AES256-SHA256:AES128-SHA:AES256-SHA:AES:CAMELLIA:DES-CBC3-SHA:!aNULL:!eNULL:!EXPORT:!DES:!RC4:!MD5:!PSK:!aECDH:!EDH-DSS-DES-CBC3-SHA:!EDH-RSA-DES-CBC3-SHA:!KRB5-DES-CBC3-SHA';

ssl_prefer_server_ciphers on;

# uniquely generated prime
ssl_dhparam DHPARAMS LOCATION;
```

## Finished Nginx Server Block Configuration

```Nginx
server {
    # listen on port 443 using ssl and spdy
    listen                  443 ssl spdy default_server;
    ssl_certificate         CERTIFICATE LOCATION;
    ssl_certificate_key     KEY LOCATION;
    ssl_session_cache       shared:SSL:20m;
    ssl_session_timeout     10m;
    ssl_protocols           TLSv1 TLSv1.1 TLSv1.2;
    # recommended secure ciphers
    ssl_prefer_server_ciphers on;
    ssl_ciphers 'ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES256-GCM-SHA384:ECDHE-ECDSA-AES256-GCM-SHA384:DHE-RSA-AES128-GCM-SHA256:DHE-DSS-AES128-GCM-SHA256:kEDH+AESGCM:ECDHE-RSA-AES128-SHA256:ECDHE-ECDSA-AES128-SHA256:ECDHE-RSA-AES128-SHA:ECDHE-ECDSA-AES128-SHA:ECDHE-RSA-AES256-SHA384:ECDHE-ECDSA-AES256-SHA384:ECDHE-RSA-AES256-SHA:ECDHE-ECDSA-AES256-SHA:DHE-RSA-AES128-SHA256:DHE-RSA-AES128-SHA:DHE-DSS-AES128-SHA256:DHE-RSA-AES256-SHA256:DHE-DSS-AES256-SHA:DHE-RSA-AES256-SHA:AES128-GCM-SHA256:AES256-GCM-SHA384:AES128-SHA256:AES256-SHA256:AES128-SHA:AES256-SHA:AES:CAMELLIA:DES-CBC3-SHA:!aNULL:!eNULL:!EXPORT:!DES:!RC4:!MD5:!PSK:!aECDH:!EDH-DSS-DES-CBC3-SHA:!EDH-RSA-DES-CBC3-SHA:!KRB5-DES-CBC3-SHA';
    # large prime for Diffie-Hellman
    ssl_dhparam DHPARAMS LOCATION;
    keepalive_timeout       70;
    # HTTP Strict Transport Security
    add_header Strict-Transport-Security "max-age=31536000; includeSubDomains";
    # key pinning
    add_header Public-Key-Pins 'pin-sha256="BASE64-KEY-INFORMATION"; pin-sha256="BASE64-KEY-INFORMATION"; max-age=5184000; includeSubDomains';

    location / {
        root /home/www/public_html/;
        index index.html;
        }
    }
```
[https-google]: http://googlewebmastercentral.blogspot.com/2014/08/https-as-ranking-signal.html
[301-google]: https://support.google.com/webmasters/answer/93633?hl=en
[301-nginx]: http://wiki.nginx.org/Pitfalls#Taxing_Rewrites
[OCSP]: http://tools.ietf.org/html/rfc2560
[OCSP-Stapling]: http://tools.ietf.org/html/rfc6066
[HSTS-ietf]: https://tools.ietf.org/html/rfc6797
[downgrade]: https://www.blackhat.com/presentations/bh-dc-09/Marlinspike/BlackHat-DC-09-Marlinspike-Defeating-SSL.pdf
[poodle]: https://www.openssl.org/~bodo/ssl-poodle.pdf
[beast-attack]: http://www.educatedguesswork.org/2011/09/security_impact_of_the_rizzodu.html
[logjam-paper]: https://weakdh.org/imperfect-forward-secrecy.pdf
[aeca-wiki]: https://en.wikipedia.org/wiki/Arms_Export_Control_Act
[NSA-spiegel]: http://www.spiegel.de/international/germany/inside-the-nsa-s-war-on-internet-security-a-1010361.html
[mozilla-SSL]: https://wiki.mozilla.org/Security/Server_Side_TLS#Recommended_configurations
[logjam-SSL]: https://weakdh.org/sysadmin.html
[base64]: https://developer.mozilla.org/en-US/docs/Web/Security/Public_Key_Pinning
[mitm-attack]: http://arstechnica.com/business/2012/02/critics-slam-ssl-authority-for-minting-cert-used-to-impersonate-sites/
