---
title: "AWS Certificate Manager TLS Certificate import"
date: 2022-08-23T11:18:30+02:00
draft: true
Summary: "When you are going to use TLS certificate with AWS CloudFront, AWS Elastic Load Balancing, AWS API Gateway or other integrated services you can use that by importing the certificate data to the AWS Certificate Manager."
---

When you are going to use TLS certificate with AWS CloudFront, AWS Elastic Load Balancing, AWS API Gateway or other integrated services you can use that by importing the certificate data to the AWS Certificate Manager.

The AWS Certificate Manager wants the following data:
- Certificate body
- Certificate private key
- Certificate chain

Here are the steps you can use to get all of this information.

## Creating TLS certificate request file

Prepare the following config file which describes what certificate you want to have.

#### **`request.conf`**
``` bash 
FQDN = # the fully qualified server (or service) name. ex.:mysite.mycompany.com
ORGNAME = # the name of your organization. ex.:mycompany
ORGUNIT = # the name of your organization unit. ex.:mycompanymydepartment
COUNTRY = # your country. ex.:US
STATE = # your state. ex.:NY
LOCALITY = # your locality. ex.:New York City
EMAIL = # administrator's email. ex.:john.doe@example.com
ALTNAMES = DNS:$FQDN

[ req ]
default_bits = 2048
default_md = sha256
prompt = no
encrypt_key = no
distinguished_name = dn
req_extensions = req_ext

[ dn ]
C = $COUNTRY
ST = $STATE
L = $LOCALITY
O = $ORGNAME
OU = $ORGUNIT
emailAddress = $EMAIL
CN = $FQDN

[ req_ext ]
subjectAltName = $ALTNAMES

```

Then run the following command to generate the csr file if you have a private key already.

``` bash 
openssl req -new -config request.conf -key mycompany_private.key -out mycompany.csr
```

Or this command if you don't have the private key and you want one to be generated.

``` bash 
openssl req -new -config request.conf -keyout mycompany_private.key -out mycompany.csr
```
You should now send this file to your TLS certificate provider.

## Parsing TLS certificate

When your TLS certificate provider respond to you you should have the '.p7b' certificate file. In combination with your private key file you now can extract all the information you need for AWS using the follwoing script:

#### **`convert.sh`**
``` bash
#!/bin/bash
# convert p7b to cer
openssl pkcs7 -print_certs -in mycompany.p7b -out mycompany.cer
# convert cer to pfx
openssl pkcs12 -export -in mycompany.cer -inkey mycompany_private.key -out mycompany.pfx
# extract data from pfx
mkdir aws
openssl pkcs12 -in mycompany.pfx -nocerts -nodes | sed -ne '/-BEGIN PRIVATE KEY-/,/-END PRIVATE KEY-/p' > aws/private.key
openssl pkcs12 -in mycompany.pfx -clcerts -nokeys | sed -ne '/-BEGIN CERTIFICATE-/,/-END CERTIFICATE-/p' > aws/certificate.cer
openssl pkcs12 -in mycompany.pfx -cacerts -nokeys -chain | sed -ne '/-BEGIN CERTIFICATE-/,/-END CERTIFICATE-/p' > aws/chain.cer
```

In a result you should have 3 files:
- private.key (copy it's content to AWS 'Certificate private key' field)
- certificate.cer (copy it's content to AWS 'Certificate bBody' field)
- chain.cer (copy it's content to AWS 'Certificate chain' field)