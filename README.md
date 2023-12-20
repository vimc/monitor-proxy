## Dockerised reverse proxy

[![Build Status](https://travis-ci.com/reside-ic/proxy-nginx.svg?branch=master)](https://travis-ci.com/reside-ic/proxy-nginx)

This repository contains support for running a `nginx` proxy in a docker container in order to secure a web application.  It makes a number of assumptions:

* You want to secure the app with TLS and redirect all http traffic to https
* You have a single service container to proxy and it is speaking http

The configuration takes as a starting point [`hint-proxy`](https://github.com/mrc-ide/hint-proxy)


## Development

To develop the proxy it is often easiest to make changes to the config directly in a running container on one of the development servers. Reload nginx config and see if the change has worked. Then update the config here and deploy it dev for a final check.

To edit the config enter the proxy container
```
docker exec -it monitor_proxy bash
```

install vim (or other) and edit the config file
```
apt-get update
apt-get install vim
vim /etc/nginx/nginx.conf
```

reload nginx config
```
nginx -s reload
```

### Configuration

Before starting we need to know what we are proxying (i.e., the name of the container on the docker network that is the main entrypoint) and what the proxy will be seen as to the outside world (the hostname, and ports for http and https).  The entrypoint takes these four values as arguments.

### SSL Certificates

The server will not start until the files `/run/proxy/certificate.pem` and `/run/proxy/key.pem` exist - you can get these into the container however you like; the proxy will poll for them and start within a second of them appearing.

### Self signed certificate

For testing it is useful to use a self-signed certificate.  These are not in any way secure.  To generate a self-signed certificate, there is a utility in the proxy container `self-signed-certificate` that will generate one on demand after receiving key components of the CSR.

There is a self-signed certificate in the repo for use in testing situations. It was generated by running (on metal):

```
./bin/self-signed-certificate ssl GB London "Imperial College" reside web-dev.dide.ic.ac.uk
```

These can be used in the container by `exec`-ing `self-signed-certificate /run/proxy` in the container while it polls for certificates.  Alternatively, to generate certificates with a custom CSR (which takes a couple of seconds) you can exec

```
self-signed-certificate GB London IC vimc montagu.vaccineimpact.org
```

### `dhparams` (Diffie-Hellman key exchange parameters)

We require a `dhparams.pem` file (see [here](https://security.stackexchange.com/questions/94390/whats-the-purpose-of-dh-parameters) for details.  To regenerate this file, run

```
./bin/dhparams ssl
```

from this directory, commit the result to git and rebuild the containers.  This takes quite a while to run (several minutes).  You can copy your own into the container at `/run/proxy/dhparams.pem` before getting the certificates in place.

### Getting a certificate from ICT

First, generate a key, use it to generate a csr and then log a ticket with ICT, sending them the csr.
  - `openssl genrsa -out my_site.key 2048`
  - `openssl req -new -key my_site.key -out my_site.csr -config path/to/openssl.cnf`

The information ICT like in the CSR is like this:
  - Country Name: `GB`
  - State or Province: `London`
  - Locality Name: `London`
  - Organization Name: `Imperial College of Science, Technology and Medicine`
  - Organizational Unit Name: `Department of Infectious Disease Epidemiology`
  - Common Name: `site_name.dide.ic.ac.uk`
  - Email Address: [BLANK]
  - Challenge Password: `a_password`
  - Optional company name: [BLANK]

Then on the [Imperial ASK](https://imperial.service-now.com/ask) site:
  - Log in with IC credentials.
  - Choose *Make a Request* and then *Security Certificate Request*.
  - Request a *Production* certificate for external use. Provide other details, and the CSR text.
  - Check your email, or revisit the ticket for updates.
  - If there is a mention of an attachment that you can't see, ring 49000 and ICT helpdesk will review visibility.

ICT will email back a set of 3 certificates in a zip;

  - `<hostname>.crt`
  - `RootCertificates/QuoVadisOVIntermediateCertificate.crt`
  - `RootCertificates/QuoVadisOVRootCertificate.crt`

We need to concatenate these three, *in the above order* to create the final certificate.  Do this manually or use `./scripts/concatenate` to do it automatically, after unziping into a directory, say `import`

```
./scripts/concatenate import/mydomain.crt import/concatenated.crt
```

We will store these in the vault: typically the vault to use is `https://vault.dide.ic.ac.uk:8200`, which you can use with

```
export VAULT_ADDR='https://vault.dide.ic.ac.uk:8200'
vault login -method=github
```

Then write into the vault as:

```
vault write /secret/domain/ssl key=@my_site.key cert=@import/concatenated.crt
```

(though the exact schema will depend on the application).  Note that this uses the private key used in the first stage.

During deployment you will need to read these keys from the vault and inject them into the container.

### Usage

```
docker run --name proxy --network yournetwork reside/proxy-nginx:reside-53 service example.com 80 443
docker cp ssl/certificate.pem proxy:/run/proxy/certificate.pem
docker cp ssl/key.pem proxy:/run/proxy/key.pem
```

The `service` here is the name and port of the http server being proxied (e.g., `myapp:8080`), as found on the network `yournetwork` (if using host networking, this could be `localhost:8080` but then you may also be exposing your http service to the wider network, which is probably undesirable).  If your service uses port `80` you may omit the port.

## License

MIT © Imperial College of Science, Technology and Medicine