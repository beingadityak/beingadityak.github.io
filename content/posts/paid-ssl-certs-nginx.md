+++
title = "Setup paid SSL Certificates for your NGINX installation"
date = 2020-10-25T19:10:42+05:30
images = []
tags = []
categories = ["ssl", "certificates", "nginx"]
draft = false
+++
![SSL Certificate](/posts/paid-ssl-certs-nginx/ssl-certificate.png)

## TLDR;

1. To generate the certs, create a CSR file with the following:
    ```
    openssl genrsa -out domain.com.key 2048
    openssl req -new -sha256 -key domain.com.key -out domain.com.cs
    ```
2. Upload the contents of the CSR to your SSL certs provider (in my case, Comodo SSL). It will create an activation file.

3. Download the same (**don’t change the name in any case!**) and upload it to your application directory (in my case, `/var/www/html`) under `.well-known/pki-validation/<PROVIDED_FILE_NAME_IN_TXT_FORMAT>`.

4. After the domain is verified, you’ll be provided with the certs in 2 parts - One will be your domain’s cert and the other will be the CA chain. Append both together and upload it to NGINX’s directory (preferably, `/etc/nginx/ssl`).

5. Configure SSL with thenginx.conf file. After that is complete, restart NGINX and allow port 443 from your firewall/security group. Check your domain and it will be having HTTPS.

## Context

Say you have created an awesome website with a popular CMS and you want to show it to the world. But when you SEO your website, as per [Google’s guidelines](https://support.google.com/webmasters/answer/7451184?hl=en) it recommends you to ensure the website is using HTTPS. You have hosted it on a server, but you have no idea about how to configure HTTPS? This guide is for you.

## Why not use Let's Encrypt?

Let’s Encrypt is a great solution! Even I use it for my websites. But this website was in a production environment and this website is supposed to be used by several thousands of users.

Since the nature of Let’s Encrypt is to be used by anyone, attackers can misuse this and can configure SSL on their websites. So there’s a good chance that many corporate environments might have blacklisted this CA from their end. Hence the choice was to use SSL certs from a commercial provider. Besides, with commercial providers, there’s an option to use Business Validation (BV)/Extended Validation (EV), a one step further than the Domain Validation (DV) provided by Let’s Encrypt ([this link](https://www.ssls.com/knowledgebase/what-is-the-difference-between-domain-validation-business-validation-and-extended-validation/) helps provide you the differences).

Let’s Encrypt certs are useful if you’re working with personal/small-scale projects or you can’t afford a commercial SSL certificate.

## Pre-requisites

To install and configure certs in NGINX, the following tools will be needed:

1. `openssl` - It is already installed in your Linux/Mac machine. You can generate the certs in your server as well, but make sure to take a backup of the RSA private key. This will be used to generate the RSA key & CSR request.

2. `nginx` - This guide is having specific steps for NGINX server, hence it should be installed in your machine.

**Note:** This process is for using wildcard SSL certificates with your server so that your multiple sub-domains will be able to use HTTPS and the certificate will become a one-time effort.

## The Process

### 1. Create the CSR (Certificate Signing Request) file

To get started with generating CSR (Certificate Signing Request), you need to create an RSA 2048-bit private key.

The reason for these many bits is for allowing more secure communication between the server and the client. If we use more number of bits, the server will take some more amount of time for finding out the key chosen by the client for performing encrypted communication.

To generate the private key, use the following command:

```
openssl genrsa -out star_domain_com.key 2048
```

This will generate an RSA key of 2048 bits. To verify whether it was an RSA key, the material of the key should start with the following:

```
-----BEGIN RSA PRIVATE KEY-----
```

This verifies that the key generated was of type RSA. Please make a note that this private key will reside on your server. **Never share the contents of your private key anywhere except in your server!**

Also, make sure to take a backup of the private key as it will be needed to renew your paid certificates.

Once your private key is generated, the next step is to generate the CSR. This will be uploaded to the Comodo SSL provider.

Create a file `csr.conf` with the following contents:

```
[ req ]
default_bits       = 2048
default_md         = sha256
default_keyfile    = star_domain_com.key
prompt             = no
encrypt_key        = no
distinguished_name = req_distinguished_name
[ req_distinguished_name ]
countryName            = "COUNTRY_CODE"
localityName           = "LOCATION"
organizationName       = "COMPANY_NAME"
organizationalUnitName = "DEPT_NAME"
commonName             = "*.domain.com"
emailAddress           = "email@domain.com"
```

Make a note of the `commonName` section. Since I’m generating a wildcard certificate, I will be providing `*.domain.com`. If I don’t use this, Comodo will throw an error saying that the Common Name must be a Wildcard one.

Run the following to generate the CSR:

```
openssl req -config csr.conf -new -key star_domain_com.key -out star_domain_com.csr
```

This will generate a `star_domain_com.csr` file. Copy its contents and paste it to Comodo’s SSL request page.

### 2. Prepare the domain for validation

After you paste the contents, there are 3 ways to verify your domain:

1. Using emails registered with the domain
2. HTTP verification or
3. DNS verification.

The DNS verification usually takes one hour and the other two are much faster. For this scenario, I’ll be using the HTTP verification.

Comodo will provide a txt file for the same. Don’t change its name as it will be used for verification purposes (To verify whether you own the domain). Upload it to the publicly accessible directory (from where your website is accessible, in my case it's `/var/www/html`) under the following directory structure:

```
/.well-known/pki-validation
# Create the above like this:
mkdir -p .well-known/pki-validation
# For my case, it will look like /var/www/html/.well-known/pki-validation
```

Under this directory structure, place the provided text so that it is accessible from your domain as follows:

```
http://domain.com/.well-known/pki-validation/<TEXT_FILE_NAME>.txt
```

Once placed, keep the file under that structure for keeping the domain validation.

### 3. Preparing the certificates

Once this is done, request validation from Comodo. After its validated, you will get a ZIP file in the email. This will contain 2 parts:

1. Your domain’s certificate and
2. the root chain.

Both of these parts are important and will be used for the installation.

To install your certs, you need to bundle these 2 parts into a single "bundle" certificate. You can do this with the following:

```
cat star_domain_com.crt star_domain_com.ca-bundle > star_domain_com_bundle.crt
```

Here, you need to make sure of the order. The first is your domain’s cert and then your subsequent CA certs. Sometimes, Comodo provides you 4 certs. For those certs, you need to follow this order to bundle them together:

```
cat domain_com.crt SectigoRSADomainValidationSecureServerCA.crt USERTrustRSAAddTrustCA.crt AddTrustExternalCARoot.crt > domain_com_bundle.crt
```

Following is the order here:

```
Domain Cert -> Domain Validation Server CA Cert -> USERTrust RSA Trust CA -> External CA Root Cert
```

This step is important so that your browser can determine the validity of the cert through the other certs chained to it. This is called **a chain of trust**. The browser will go through the cert chain to determine the root CA and with the root CA cert, it will be able to determine that the provided domain cert is correct and valid.

### 4. Installing the certificates in your server

The last step of this workflow is to place the certs and the private key (which we used to generate the CSR) in a convenient location so that NGINX can read this and enable it to serve traffic using HTTPS.

1. Make an archive of the bundled certs and the private key and copy the archive to the server.
2. Extract the archive to `/etc/nginx/ssl` location.
3. Enable SSL using the following configuration (Use `/etc/nginx/conf.d/domain.com.conf` for keeping the configurations clean):
    ```
    server {
        listen 80;
        server_name example.com; 
        return 301 https://$host$request_uri;
    }
    server {
        server_name example.com;
        listen 443 ssl default_server;
        ssl on;
        ssl_certificate /etc/nginx/ssl/star_domain_com_bundle.crt;
        ssl_certificate_key /etc/nginx/ssl/star_domain_com.key;
        ssl_protocols TLSv1 TLSv1.1 TLSv1.2;
    }
    ```
With this configuration, you can run HTTPS on your website as well as redirect HTTP requests to HTTPS. Please make sure that you replace the server_name directive to your domain name.

4. Reload NGINX to see the changes.

## Renewal Instructions

For every renewal, you need to create new CSRs and upload the renewed certs to `/etc/nginx/ssl` for keeping customers’ traffic secure.