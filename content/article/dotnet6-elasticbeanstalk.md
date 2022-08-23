---
title: "Deploying dotnet6 app on AWS ElasticBeanstalk Graviton2 with HTTPS termination on EC2 instance"
date: 2022-08-22T10:20:06+02:00
draft: false
Summary: "On one of my project it was a requirement to reduce the ElasticBeanstalk hosting costs to minimum for dotnet application. The app is a kind of dashboard with limited audience and nothing heavy inside. I decided to use Graviton2 instances (ARM processors on board) in order to acheave best speed/cost ratio."
---

On one of my project it was a requirement to reduce the ElasticBeanstalk hosting costs to minimum for dotnet application. The app is a kind of dashboard with limited audience and nothing heavy inside. I decided to use Graviton2 instances (ARM processors on board) in order to acheave best speed/cost ratio.

Table of content:
1.   Build dotnet app on Windows for arm linux environment
2.   Configure app to run on Linux environment in Amazon
3.   Configure app to terminate HTTPS connections at the instance
4.   Configure HTTP to HTTPS redirection
5.   Pack deployment package

## Build dotnet app for arm linux environment

This is as easy as:

``` bash
dotnet publish myproject.csproj --configuration Release --runtime linux-arm64 --output publish
```

That's it. the 'publish' directory will contain your published app (not ready yet for Amazon EB).

## Configure app to run on Linux environment in Amazon

You need to create [Procfile](https://docs.aws.amazon.com/elasticbeanstalk/latest/dg/dotnet-linux-procfile.html) for your application. It's contect should look like this:

#### **`Procfile`**
```
web: dotnet ./myproject.dll
```

## Configure app to terminate HTTPS connections at the instance

First you need to create 'https-instance-single.config' file that configures Amazon security group and port mapping. [[reference]](https://docs.aws.amazon.com/elasticbeanstalk/latest/dg/https-singleinstance.html).

#### **`https-instance-single.config`**
``` config
Resources:
  sslSecurityGroupIngress: 
    Type: AWS::EC2::SecurityGroupIngress
    Properties:
      GroupId: {"Fn::GetAtt" : ["AWSEBSecurityGroup", "GroupId"]}
      IpProtocol: tcp
      ToPort: 443
      FromPort: 443
      CidrIp: 0.0.0.0/0
```

Then you need to create 'https-instance.config' file that describes your TLS certificates and where shoud they be placed on EB instance. [[reference]](https://docs.aws.amazon.com/elasticbeanstalk/latest/dg/https-singleinstance-dotnet-linux.html).

#### **`https-instance.config`**
``` config
files:
  /etc/pki/tls/certs/server.crt:
    content: |
      -----BEGIN CERTIFICATE-----
      ..........................
      -----END CERTIFICATE-----
      
  /etc/pki/tls/certs/server.key:
    content: |      
      -----BEGIN RSA PRIVATE KEY-----
      ..............................
      -----END RSA PRIVATE KEY-----

container_commands:
  01restart_nginx:
    command: "systemctl restart nginx"
```

And 'https.conf' file to configure NGINX Web server where to find TLS certificate and how to use it.

#### **`https.conf`**
``` config
# HTTPS server

server {
    listen       443 ssl;
    server_name  localhost;
    
    ssl_certificate      /etc/pki/tls/certs/server.crt;
    ssl_certificate_key  /etc/pki/tls/certs/server.key;
    
    ssl_session_timeout  5m;
    
    ssl_protocols  TLSv1 TLSv1.1 TLSv1.2;
    ssl_prefer_server_ciphers   on;
    
    location / {
        proxy_pass  http://localhost:5000;
        proxy_set_header   Connection "";
        proxy_http_version 1.1;
        proxy_set_header        Host            $host;
        proxy_set_header        X-Real-IP       $remote_addr;
        proxy_set_header        X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header        X-Forwarded-Proto https;
    }
}
```

## Configure HTTP to HTTPS redirection

You need to create '00_application.conf' file that defines http redirects [[reference]](https://docs.aws.amazon.com/elasticbeanstalk/latest/dg/configuring-https-httpredirect.html) + [[StackOverflow thread]](https://stackoverflow.com/questions/51900577/redirect-elastic-beanstalk-http-requests-to-https-with-nginx).

#### **`00_application.conf`**
``` config
location / {
     set $redirect 0;
     if ($http_x_forwarded_proto != "https") {
       set $redirect 1;
     }
     if ($http_user_agent ~* "ELB-HealthChecker") {
       set $redirect 0;
     }
     if ($redirect = 1) {
       return 301 https://$host$request_uri;
     }   
 
     proxy_pass          http://127.0.0.1:5000;
     proxy_http_version  1.1;
 
     proxy_set_header    Connection          $connection_upgrade;
     proxy_set_header    Upgrade             $http_upgrade;
     proxy_set_header    Host                $host;
     proxy_set_header    X-Real-IP           $remote_addr;
     proxy_set_header    X-Forwarded-For     $proxy_add_x_forwarded_for;
}
```

## Pack deployment package

Put all together:

``` bash
publish
|───..Yout published dotnet app..
assets
│   Procfile
├───.ebextensions
│       https-instance-single.config
│       https-instance.config
└───.platform
    └───nginx
        └───conf.d
            │   https.conf
            └───elasticbeanstalk
                    00_application.conf
```

Finally you need to prepare the deployment artifact. If you are using Windows then you can't use powershell 'Compress-Archive' to zip directories because it uses backslashes as path separators. [SuperUser thread about this](https://superuser.com/questions/1382839/zip-files-expand-with-backslashes-on-linux-no-subdirectories). We are going to use [7Zip](https://www.7-zip.org/).

``` bash
."C:\Program Files\7-Zip\7z.exe" a -tzip publish.zip ".\publish\*" ".\assets\*"
```

## Costs

AWS t4g.nano costs about $3/month.