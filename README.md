#Let's Encrypt Without Sudo

The [Let's Encrypt](https://letsencrypt.org/) initiative is a fantastic program
that offers **free** https certificates! However, the one catch is that you need
to use their command program to get a free certificate. The default instructions
all assume that you will run it on your your server as root, and that it will
edit your apache/nginx config files.

I love the Let's Encrypt devs dearly, but there's no way I'm going to trust
their script to run on my server as root, be able to edit my server configs, and
have access to my private keys. I'd just like the free ssl certificate, please.

So I made a script that does that. You generate your private key and certificate
signing request (CSR) like normal, then run `sign_csr.py` with your CSR to get
it signed. The script goes through the [ACME protocol](https://github.com/letsencrypt/acme-spec)
with the Let's Encrypt certificate authority and outputs the signed certificate
to stdout.

This script doesn't know or ask for your private key, and it doesn't need to be
run on your server. There are some parts of the ACME protocol that require your
private key and access to your server. For those parts, this script prints out
very minimal commands for you to run to complete the requirements. There is only
one command that needs to be run as root on your server and it is a very simple
python https server that you can inspect for yourself before you run it.

##Table of Contents

* [Donate](#donate)
* [Prerequisites](#prerequisites)
* Signing script
    * [How to use the signing script](#how-to-use-the-signing-script)
    * [Example use of the signing script](#example-use-of-the-signing-script)
    * [How to use the signed https certificate](#how-to-use-the-signed-https-certificate)
    * [Demo](#demo)
* Revocation script
    * [How to use the revocation script](#how-to-use-the-revocation-script)
    * [Example use of the revocation script](#example-use-of-the-revocation-script)
* [Alternative: Official Let's Encrypt Client](#alternative-official-lets-encrypt-client)
* [Feedback/Contributing](#feedbackcontributing)

##Donate

If this script is useful to you, please donate to the EFF. I don't work there,
but they do fantastic work.

[https://eff.org/donate/](https://eff.org/donate/)

##Prerequisites

* openssl
* python

##How to use the signing script

First, you need to generate an user account key for Let's Encrypt.
This is the key that you use to register with Let's Encrypt. If you
already have user account key with Let's Encrypt, you can skip this
step.

```sh
openssl genrsa 4096 > user.key
openssl rsa -in user.key -pubout > user.pub
```

Second, you need to generate the domain key and a certificate request.
This is the key that you will get signed for free for your domain (replace
"example.com" with the domain you own). If you already have a domain key
and CSR for your domain, you can skip this step.

```sh
#Create a CSR for example.com
openssl genrsa 4096 > domain.key
openssl req -new -sha256 -key domain.key -subj "/CN=example.com" > domain.csr

#Alternatively, if you want both example.com and www.example.com
openssl genrsa 4096 > domain.key
openssl req -new -sha256 -key domain.key -subj "/" -reqexts SAN -config <(cat /etc/ssl/openssl.cnf <(printf "[SAN]\nsubjectAltName=DNS:example.com,DNS:www.example.com")) > domain.csr
```

Third, you run the script using python and passing in the path to your user
account public key and the domain CSR. The paths can be relative or absolute.
By default the script will ask you to start a webserver on port 80.  If you
already have one, use the `--file-based` option instead.

```sh
python sign_csr.py --public-key user.pub domain.csr > signed.crt
```

When you run the script, it will ask you do do some manual commands. It has to
ask you to do these because it doesn't know your private key or have access to
your server. You can edit the manual commands to fit your situation (e.g. if
your sudo user is different or private key is in a different location).

NOTE: When the script asks you to run these manual commands, you need to run
them in a separate terminal window. You need to keep the script open while you
run them. They sign temporary test files that the script created, so if you exit
or continue the script before you run the commands, those test files will be
destroyed before they can be used correctly (and you'll have to run the script
again).

The `*.json` and `*.sig` files are temporary files automatically generated by
the script and will be destroyed when the script stops. They only contain the
protocol requests and signatures. They do NOT contain your private keys
because this script does not have access to your private keys.

###Help text
```
user@hostname:~$ python sign_csr.py --help
usage: sign_csr.py [-h] -p PUBLIC_KEY [-e EMAIL] csr_path

Get a SSL certificate signed by a Let's Encrypt (ACME) certificate authority and
output that signed certificate. You do NOT need to run this script on your
server and this script does not ask for your private keys. It will print out
commands that you need to run with your private key or on your server as root,
which gives you a chance to review the commands instead of trusting this script.

NOTE: YOUR ACCOUNT KEY NEEDS TO BE DIFFERENT FROM YOUR DOMAIN KEY.

Prerequisites:
* openssl
* python

Example: Generate an account keypair, a domain key and csr, and have the domain csr signed.
--------------
$ openssl genrsa 4096 > user.key
$ openssl rsa -in user.key -pubout > user.pub
$ openssl genrsa 4096 > domain.key
$ openssl req -new -sha256 -key domain.key -subj "/CN=example.com" > domain.csr
$ python sign_csr.py --public-key user.pub domain.csr > signed.crt
--------------

positional arguments:
  csr_path              path to your certificate signing request

optional arguments:
  -h, --help            show this help message and exit
  -p PUBLIC_KEY, --public-key PUBLIC_KEY
                        path to your account public key
  -e EMAIL, --email EMAIL
                        contact email, default is webmaster@<shortest_domain>
  -f, --file-based      if set, a file-based response is used
user@hostname:~$
```

##Example use of the signing script

###Commands (what you do in your main terminal window)
```
user@hostname:~$ openssl genrsa 4096 > user.key
Generating RSA private key, 4096 bit long modulus
.............................................................................................................................................................................++
....................................................++
e is 65537 (0x10001)
user@hostname:~$ openssl rsa -in user.key -pubout > user.pub
writing RSA key
user@hostname:~$ openssl genrsa 4096 > domain.key
Generating RSA private key, 4096 bit long modulus
.................................................................................................................................................................................++
...........................................++
e is 65537 (0x10001)
user@hostname:~$ openssl req -new -sha256 -key domain.key -subj "/CN=letsencrypt.daylightpirates.org" > domain.csr
user@hostname:~$ python sign_csr.py --public-key user.pub domain.csr > signed.crt
Reading pubkey file...
Found public key!
Reading csr file...
Found domains letsencrypt.daylightpirates.org
STEP 1: What is your contact email? (webmaster@letsencrypt.daylightpirates.org) daniel@roesler.cc
Building request payloads...
STEP 2: You need to sign some files (replace 'user.key' with your user private key).

openssl dgst -sha256 -sign user.key -out register_KN2ihH.sig register_ABUO4T.json
openssl dgst -sha256 -sign user.key -out domain_BbpWG4.sig domain_rSKa5G.json
openssl dgst -sha256 -sign user.key -out challenge_fo6_ib.sig challenge_e3gHzd.json
openssl dgst -sha256 -sign user.key -out cert_36OUdW.sig cert_3IZULZ.json

Press Enter when you've run the above commands in a new terminal window...
Registering daniel@roesler.cc...
Already registered. Skipping...
Requesting challenges for letsencrypt.daylightpirates.org...
STEP 3: You need to sign some more files (replace 'user.key' with your user private key).

openssl dgst -sha256 -sign user.key -out response_ATE3Yu.sig response_P87LMt.json

Press Enter when you've run the above commands in a new terminal window...
STEP 4: You need to run this command on letsencrypt.daylightpirates.org (don't stop the python command until the next step).

sudo python -c "import BaseHTTPServer; \
    h = BaseHTTPServer.BaseHTTPRequestHandler; \
    h.do_GET = lambda r: r.send_response(200) or r.end_headers() or r.wfile.write('{\"header\": {\"alg\": \"RS256\"}, \"protected\": \"eyJhbGciOiAiUlMyNTYifQ\", \"payload\": \"ewogICAgInRscyI6IGZhbHNlLCAKICAgICJ0b2tlbiI6ICJkbzVaWkMwMHVwZmNFN0tjeEhzOGNyS2FNaE02UFdBdTMtMnVwZ00zRG00IiwgCiAgICAidHlwZSI6ICJzaW1wbGVIdHRwIgp9\", \"signature\": \"Gp5V68da_XdC96piXs1YOhrv4USOQBNnhIL-CMmxvKSigmxAJ8z00xsgWS6nsYD8LPpMVa3GkXhb10qfbymPiWhtMpMYZ31kMLFwgpHrY9xkiNP-WK9Zljz6L-WAzxCOmF1Ov71z_75iEJij86E2f9EmTjDlmDmGAjP9lziII42uyyjjIZg9claU1GtFZUrfXd-uNHHEGHFUpoyLHQcyWCP1T04Xx4q4dY51VeOJNOmIv9csIjkbOma7EqFMAHwYAplAUE45FQ5N9lJvpymD49BoEgQj_kjH-UPnxO3q0QB0i-MJJCiwQYAhMKV618jV9rNE181zJ1FRkX48knMzqoE4oG3yEFUg2D_vAdFG3VCuotnuxrZ7BEzDPWyEm0z8XakxWQW-xHSADtKWRr1qsQCy7qVsoAKnVFQ_1b4rAzET1YfrmhSH4MVhMB5n9tOnjtPQ0OsJVbf0oVLh5AC1rbXe68weOQExDVJgsk56x3FvvwrmdaLe2TnbPJmzpkYUf1OK88e8KmhVYb34veuY1luDOBJQyQ9fOAGZC0F-g7SpWg1lp3hQzf5enkycHMK-fNAfFH7t1m1Ej_CvUuxfBVhI0W8ANpFWL4r8PxTZaZzE6NO38MYgB9nrICiKJuuTQQbsXdjOm22QuxrG1XpWA-vQCtbk-L891Ko6MdAUMzQ\"}'); \
    s = BaseHTTPServer.HTTPServer(('0.0.0.0', 80), h); \
    s.serve_forever()"

Press Enter when you've got the python command running on your server...
Requesting verification for letsencrypt.daylightpirates.org...
Waiting for letsencrypt.daylightpirates.org challenge to pass...
Passed letsencrypt.daylightpirates.org challenge!
Requesting signature...
Certificate signed!
You can stop running the python command on your server (Ctrl+C works).
user@hostname:~$ cat signed.crt
-----BEGIN CERTIFICATE-----
MIIGJTCCBQ2gAwIBAgISATBRUGjFwTtjF4adpF7zd/5qMA0GCSqGSIb3DQEBCwUA
MEoxCzAJBgNVBAYTAlVTMRYwFAYDVQQKEw1MZXQncyBFbmNyeXB0MSMwIQYDVQQD
ExpMZXQncyBFbmNyeXB0IEF1dGhvcml0eSBYMTAeFw0xNTEwMjQwOTU4MDBaFw0x
NjAxMjIwOTU4MDBaMCoxKDAmBgNVBAMTH2xldHNlbmNyeXB0LmRheWxpZ2h0cGly
YXRlcy5vcmcwggIiMA0GCSqGSIb3DQEBAQUAA4ICDwAwggIKAoICAQC2Ac7twhMz
AxreQxmlY0gBq20zrriMOCLTwwdJ3sfv9bNxo+iG7eidu9imLI0FNjZkxtpyJeG/
+4OnvTgChHiTEKtD0Q3SoeSOu3Bl73d4bVBfTsvj0yEoMrF4Y89VvqbH7HP+2evv
Uraj2Qv0EUor3KAsOJW4hiSQedmz69+3IVZHWdpyYTtC1HjO9C5DqPgD7hlrtRrP
k0SL4j048NIiDvMm36pzn/UM+HxuavVxIyQ7BigDk7Hev6jXH2BqQk0ADtR0CycI
nJeS5gk+i6ImDeOsrhPrXvub02aRbol/paoSknskAOJKe4628dd873QfMXnQz1JT
aggaFQA1S8M2DY9l574/gOH39BudXdvOGzln7MeDJoi7Tybih2FJJbj8tQPV2zwh
ArbKLHPJibM1HP8jc7QQcrWnNf3H2N5FhP8uvEVchdYk3zV2tJPqlQnsHctOjNrV
18WRsl+JpUNLclRWQ3JLYZL+waIaJvsAsjp58J3XK1PI1s7QPuJpI3u7hlu4zz2e
TMF8OqAEy+rkHML5j+ncB+ctxhgNgirwpCUQ3NL9rslte0OmO+kzjrVfJ7o5D6zt
Hn5xg2WTgNoCdXbIruEzC43SqkPIH8VeFkzjPCqGajQsXXmdbDyoNkJ+SK0Fz0hI
3alW4kaOSe0aeto22sKtOjsIy7GF6qDw4QIDAQABo4ICIzCCAh8wDgYDVR0PAQH/
BAQDAgWgMB0GA1UdJQQWMBQGCCsGAQUFBwMBBggrBgEFBQcDAjAMBgNVHRMBAf8E
AjAAMB0GA1UdDgQWBBSpGhk6yOALnLPWzrncMA/wnd6nNzAfBgNVHSMEGDAWgBSo
SmpjBH3duubRObemRWXv86jsoTBwBggrBgEFBQcBAQRkMGIwLwYIKwYBBQUHMAGG
I2h0dHA6Ly9vY3NwLmludC14MS5sZXRzZW5jcnlwdC5vcmcvMC8GCCsGAQUFBzAC
hiNodHRwOi8vY2VydC5pbnQteDEubGV0c2VuY3J5cHQub3JnLzAqBgNVHREEIzAh
gh9sZXRzZW5jcnlwdC5kYXlsaWdodHBpcmF0ZXMub3JnMIIBAAYDVR0gBIH4MIH1
MAoGBmeBDAECATAAMIHmBgsrBgEEAYLfEwEBATCB1jAmBggrBgEFBQcCARYaaHR0
cDovL2Nwcy5sZXRzZW5jcnlwdC5vcmcwgasGCCsGAQUFBwICMIGeDIGbVGhpcyBD
ZXJ0aWZpY2F0ZSBtYXkgb25seSBiZSByZWxpZWQgdXBvbiBieSBSZWx5aW5nIFBh
cnRpZXMgYW5kIG9ubHkgaW4gYWNjb3JkYW5jZSB3aXRoIHRoZSBDZXJ0aWZpY2F0
ZSBQb2xpY3kgZm91bmQgYXQgaHR0cHM6Ly9sZXRzZW5jcnlwdC5vcmcvcmVwb3Np
dG9yeS8wDQYJKoZIhvcNAQELBQADggEBADQ2nWJa0jSOgStC7luKLmNOiNZTbiYP
ITFetj6WpRIsAHwz3vTwDIWFtczrhksWRTU9mCIwaxtqflZrirc3mE6jKugeSUHr
1yqTXZ097rDNAnMvUtvoET/UBkAU+gUDn8zRFtKOePuWX7P8qHq8QqjNqMC0vb5s
ncyFqSSZl1j9e5l+Kpj/GeTCwkwck5U75Ry44kPbnu5JLd70P724gBnyEi6IxXHB
txXZEUmI0R1Ee3Kw/5N6JfeWNE1KEmM47VVFomRitruxBj9nlXtIILvkPCTWkDua
pr1OmFi/rUcaHw+Txbs8aBmZEBkxy9HPSfgqqlYqEd0ipGqFtqaFJEI=
-----END CERTIFICATE-----
user@hostname:~$
```

###Manual Commands (the stuff the script asked you to do in a 2nd terminal)
```
#first set of signed files
user@hostname:~$ openssl dgst -sha256 -sign user.key -out register_KN2ihH.sig register_ABUO4T.json
user@hostname:~$ openssl dgst -sha256 -sign user.key -out domain_BbpWG4.sig domain_rSKa5G.json
user@hostname:~$ openssl dgst -sha256 -sign user.key -out challenge_fo6_ib.sig challenge_e3gHzd.json
user@hostname:~$ openssl dgst -sha256 -sign user.key -out cert_36OUdW.sig cert_3IZULZ.json
user@hostname:~$

#second set of signed files
user@hostname:~$ openssl dgst -sha256 -sign user.key -out response_ATE3Yu.sig response_P87LMt.json
user@hostname:~$
```

###Server Commands (the stuff the script asked you to do on your server)
```
ubuntu@letsencrypt.daylightpirates.org:~$ sudo python -c "import BaseHTTPServer; \
>     h = BaseHTTPServer.BaseHTTPRequestHandler; \
>     h.do_GET = lambda r: r.send_response(200) or r.end_headers() or r.wfile.write('{\"header\": {\"alg\": \"RS256\"}, \"protected\": \"eyJhbGciOiAiUlMyNTYifQ\", \"payload\": \"ewogICAgInRscyI6IGZhbHNlLCAKICAgICJ0b2tlbiI6ICJkbzVaWkMwMHVwZmNFN0tjeEhzOGNyS2FNaE02UFdBdTMtMnVwZ00zRG00IiwgCiAgICAidHlwZSI6ICJzaW1wbGVIdHRwIgp9\", \"signature\": \"Gp5V68da_XdC96piXs1YOhrv4USOQBNnhIL-CMmxvKSigmxAJ8z00xsgWS6nsYD8LPpMVa3GkXhb10qfbymPiWhtMpMYZ31kMLFwgpHrY9xkiNP-WK9Zljz6L-WAzxCOmF1Ov71z_75iEJij86E2f9EmTjDlmDmGAjP9lziII42uyyjjIZg9claU1GtFZUrfXd-uNHHEGHFUpoyLHQcyWCP1T04Xx4q4dY51VeOJNOmIv9csIjkbOma7EqFMAHwYAplAUE45FQ5N9lJvpymD49BoEgQj_kjH-UPnxO3q0QB0i-MJJCiwQYAhMKV618jV9rNE181zJ1FRkX48knMzqoE4oG3yEFUg2D_vAdFG3VCuotnuxrZ7BEzDPWyEm0z8XakxWQW-xHSADtKWRr1qsQCy7qVsoAKnVFQ_1b4rAzET1YfrmhSH4MVhMB5n9tOnjtPQ0OsJVbf0oVLh5AC1rbXe68weOQExDVJgsk56x3FvvwrmdaLe2TnbPJmzpkYUf1OK88e8KmhVYb34veuY1luDOBJQyQ9fOAGZC0F-g7SpWg1lp3hQzf5enkycHMK-fNAfFH7t1m1Ej_CvUuxfBVhI0W8ANpFWL4r8PxTZaZzE6NO38MYgB9nrICiKJuuTQQbsXdjOm22QuxrG1XpWA-vQCtbk-L891Ko6MdAUMzQ\"}'); \
>     s = BaseHTTPServer.HTTPServer(('0.0.0.0', 80), h); \
>     s.serve_forever()"
66.133.109.36 - - [24/Oct/2015 06:58:10] "GET /.well-known/acme-challenge/do5ZZC00upfcE7KcxHs8crKaMhM6PWAu3-2upgM3Dm4 HTTP/1.1" 200 -
^CTraceback (most recent call last):
  File "<string>", line 1, in <module>
  File "/usr/lib/python2.7/SocketServer.py", line 236, in serve_forever
    poll_interval)
  File "/usr/lib/python2.7/SocketServer.py", line 155, in _eintr_retry
    return func(*args)
KeyboardInterrupt
ubuntu@letsencrypt.daylightpirates.org:~$
```

##How to use the signed https certificate

The signed https certificate that is output by this script can be used along
with your private key to run an https server. You just securely transfer (using
`scp` or similar) the private key and signed certificate to your server, then
include them in the https settings in your web server's configuration. Here's an
example on how to configure an nginx server:

```
#NOTE: For nginx, you need to append the Let's Encrypt intermediate cert to your cert
user@hostname:~$ wget https://letsencrypt.org/certs/lets-encrypt-x3-cross-signed.pem
user@hostname:~$ cat signed.crt lets-encrypt-x3-cross-signed.pem > chained.pem
```

```nginx
server {
    listen 443;
    server_name letsencrypt.daylightpirates.org;
    ssl on;
    ssl_certificate chained.pem;
    ssl_certificate_key domain.key;
    ssl_session_timeout 5m;
    ssl_protocols TLSv1 TLSv1.1 TLSv1.2;
    ssl_ciphers ECDHE-RSA-AES256-GCM-SHA384:ECDHE-RSA-AES128-GCM-SHA256:DHE-RSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-SHA384:ECDHE-RSA-AES128-SHA256:ECDHE-RSA-AES256-SHA:ECDHE-RSA-AES128-SHA:DHE-RSA-AES256-SHA:DHE-RSA-AES128-SHA;
    ssl_session_cache shared:SSL:50m;
    ssl_dhparam /etc/nginx/server.dhparam;
    ssl_prefer_server_ciphers on;

    location / {
        return 200 'Let\'s Encrypt Example: https://github.com/diafygi/letsencrypt-nosudo';
        add_header Content-Type text/plain;
    }
}
```

##Demo

Here's a website that is using a certificate signed using `sign_csr.py`:

[https://letsencrypt.daylightpirates.org/](https://letsencrypt.daylightpirates.org/)

##How to use the revocation script

First, you will need to the user account key for Let's Encrypt that was used
when the certifacate was signed.

Second, you will need the PEM encoded signed certificate that was produced by
`sign_csr.py`.

Third, you run the script using python and passing in the path to your user
account public key and the signed domain certificate. The paths can be relative
or absolute.  If you wish to give the script access to your user private key, it
can accept that as an optional argument.

```sh
python revoke_crt.py --public-key user.pub domain.crt
```

When you run the script, it will ask you do one manual signature.  It has to ask you
to do these because it doesn't know your private key. You can edit the manual
commands to fit your situation (e.g. if your private key is in a different
location).

NOTE: When the script asks you to run these manual commands, you need to run
them in a separate terminal window. You need to keep the script open while you
run them. They sign temporary test files that the script created, so if you exit
or continue the script before you run the commands, those test files will be
destroyed before they can be used correctly (and you'll have to run the script
again).

The `*.json` and `*.sig` files are temporary files automatically generated by
the script and will be destroyed when the script stops. They only contain the
protocol requests and signatures. They do NOT contain your private keys
because this script does not have access to your private keys.

###Help text
```
user@hostname:~$ python revoke_crt.py --help
usage: revoke_crt.py [-h] -p PUBLIC_KEY [-r PRIVATE_KEY] crt_path

Get a SSL certificate revoked by a Let's Encrypt (ACME) certificate authority.
You do NOT need to run this script on your server and this script does not ask
for your private keys. It will print out commands that you need to run with
your private key, which gives you a chance to review the commands instead of
trusting this script.

NOTE: YOUR PUBLIC KEY NEEDS TO BE THE SAME KEY USED TO ISSUE THE CERTIFICATE.

Prerequisites:
* openssl
* python

Example:
--------------
$ python revoke_crt.py --public-key user.pub domain.crt
--------------

positional arguments:
  crt_path              path to your signed certificate

optional arguments:
  -h, --help            show this help message and exit
  -p PUBLIC_KEY, --public-key PUBLIC_KEY
                        path to your account public key
user@hostname:~$
```

##Example use of the revocation script

###Commands (what you do in your main terminal window)
```
user@hostname:~$ python revoke_crt.py --public-key user.pub domain.crt
Reading pubkey file...
Found public key!
STEP 1: You need to sign a file (replace 'user.key' with your user private key)

openssl dgst -sha256 -sign user.key -out revoke_Z5Qxj3.sig revoke_TKSK9w.json

Press Enter when you've run the above command in a new terminal window...
Requesting revocation...
Certificate revoked!
user@hostname:~$
```

###Manual Command (the stuff the script asked you to do in a 2nd terminal)
```
#signed files
user@hostname:~$ openssl dgst -sha256 -sign user.key -out revoke_Z5Qxj3.sig revoke_TKSK9w.json
```

##Alternative: Official Let's Encrypt Client

After I released this script, Let's Encrypt added a manual authenticator to
allow the Let's Encrypt client to not have to be run on your server. Hooray!
However, the Let's Encrypt client still has access to your user account private
keys, so please be aware of that. Anyway, check out the comment on issue
[#5](https://github.com/diafygi/letsencrypt-nosudo/issues/5#issuecomment-117283651)
to see how to use the manual authenticator in the official Let's Encrypt client.

```
./letsencrypt-auto --email diafygi@gmail.com --text --authenticator manual --work-dir /tmp/work/ --config-dir /tmp/config/ --logs-dir /tmp/logs/ auth --cert-path /tmp/certs/ --chain-path /tmp/chains/ --csr ~/Desktop/domain.csr
```

##Feedback/Contributing

I'd love to receive feedback, issues, and pull requests to make this script
better. The script itself, `sign_csr.py`, is less than 500 lines of code, so
feel free to read through it! I tried to comment things well and make it crystal
clear what it's doing.

For example, it currently can't do any ACME challenges besides 'http-01'. Maybe
someone could do a pull request to add more challenge compatibility?


