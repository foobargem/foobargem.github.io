---
title: "IAM role and metadata service for EC2"
date: 2023-09-16T08:14:48+09:00
author: foobargem
tags: ["aws", "iam"]
---

> **_TL;DR:_** EC2 instance 의 IAM role 은 metadata service 가 필요하다

EC2 instance 에서 S3 같은 다른 리소스에 접근하기 위해서 자격증명(Credentials) 을 사용해야 한다.
[몇가지 방법들](https://docs.aws.amazon.com/cli/latest/userguide/cli-configure-files.html)이 있지만
instance profile 로 IAM role 을 만들고, EC2 instance 에 IAM role 을 할당하는 것이 일반적인 방법이다.

최근에 TLS 인증서 교체 작업중 EC2 instance 의 IAM role 이 동작할때 metadata service 의 관련이 있는것을 알게 되어서 이 글을 작성한다.

새로운 TLS 인증서 파일은 S3 bucket 에 업로드를 해두었다.
그리고 IAM role 이 적용된 EC2 instance 에서 awscli 로 인증서 object 복사를 시도하니 `Unable to locate credentials.` 라는 오류가 발생을 했다.

```
$ aws s3 --recursive cp s3://foobargem-bucket/cert/foobargemd.dev/ /etc/cert/

Unable to locate credentials. You can configure credentials by running "aws configure".
```

원인을 추적하기 위해 `--debug` 옵션을 붙여보았고, 아래처럼 metadata api 와 통신이 실패한것이 원인임을 알수 있었다.

```
...
$ aws --debug s3 --recursive cp s3://foobargem-bucket/cert/foobargemd.dev/ /etc/cert/
...
2023-09-15 23:40:41,748 - MainThread - botocore.hooks - DEBUG - Event session-initialized: calling handler <function inject_assume_role_provider_cache at 0x7fc6fc33d940>
2023-09-15 23:40:41,749 - MainThread - botocore.utils - DEBUG - IMDS ENDPOINT: http://169.254.169.254/
...
2023-09-15 23:40:41,754 - MainThread - botocore.credentials - DEBUG - Looking for credentials via: iam-role
2023-09-15 23:40:41,755 - MainThread - urllib3.connectionpool - DEBUG - Starting new HTTP connection (1): 169.254.169.254:80
2023-09-15 23:40:41,756 - MainThread - urllib3.connectionpool - DEBUG - http://169.254.169.254:80 "PUT /latest/api/token HTTP/1.1" 200 56
2023-09-15 23:40:41,757 - MainThread - urllib3.connectionpool - DEBUG - Resetting dropped connection: 169.254.169.254
2023-09-15 23:40:41,758 - MainThread - urllib3.connectionpool - DEBUG - http://169.254.169.254:80 "GET /latest/meta-data/iam/security-credentials/ HTTP/1.1" 404 339
2023-09-15 23:40:41,758 - MainThread - botocore.utils - DEBUG - Metadata service returned non-200 response with status code of 404 for url: http://169.254.169.254/latest/meta-data/iam/security-credentials/, content body: b'<?xml version="1.0" encoding="iso-8859-1"?>\n<!DOCTYPE html PUBLIC "-//W3C//DTD XHTML 1.0 Transitional//EN"\n\t\t "http://www.w3.org/TR/xhtml1/DTD/xhtml1-transitional.dtd">\n<html xmlns="http://www.w3.org/1999/xhtml" xml:lang="en" lang="en">\n <head>\n  <title>404 - Not Found</title>\n </head>\n <body>\n  <h1>404 - Not Found</h1>\n </body>\n</html>\n'
2023-09-15 23:40:41,758 - MainThread - botocore.utils - DEBUG - Max number of attempts exceeded (1) when attempting to retrieve data from metadata service.
...
```

[AWS 문서](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/iam-roles-for-amazon-ec2.html#instance-metadata-security-credentials)를 읽어보니 `http://169.254.169.254/latest/meta-data/iam/security-credentials/{IAM_ROLE_NAME}` 을 통해서 IAM role 을 가져온다는 것을 알수 있었다.

EC2 instance 의 metadata service 를 enable (with IMDSv2:required) 로 변경한뒤 IAM role 로 자격증명이 되는것을 확인했다.


## 참고
* https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/iam-roles-for-amazon-ec2.html
