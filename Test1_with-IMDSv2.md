## Configure the instance metadata service Version 2 (IDMSV2) 

AWS documentation, how to enable IMDSv2:
https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/configuring-instance-metadata-service.html

A nice write-up of SSRF vulnerability mitigation with IMDSV2 (December, 2019)
https://www.cyberbit.com/blog/uncategorized/aws-imds-v2-secures-ssrf-vulnerability/


<HR>

Before: Verify remote access to query instance credentials through metadata


```
root@HOSTNAME:~# curl -s http://100.25.77.36/latest/meta-data/iam/security-credentials/ -H 'Host:169.254.169.254'  
cg-banking-WAF-Role-cgidatyedvvzzy
```

Use role to retrieve access keys:
```
root@HOSTNAME:~# curl -s http://100.25.77.36/latest/meta-data/iam/security-credentials/cg-banking-WAF-Role-cgidatyedvvzzy -H 'Host:169.254.169.254'  
{
  "Code" : "Success",
  "LastUpdated" : "2020-01-20T20:38:19Z",
  "Type" : "AWS-HMAC",
  "AccessKeyId" : "xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx",
  "SecretAccessKey" : "xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx",
  "Token" : "xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxtsg==",
  "Expiration" : "2020-01-21T02:56:00Z"
}root@HOSTNAME:~# 
```

<HR>


Query metadata v2 (on the local instance) to verify IDMDSV2 is working: 
 
1) Set Token: 

```
     root@ip-10-10-10-89:~# TOKEN=`curl -X PUT "http://169.254.169.254/latest/api/token" -H "X-aws-ec2-metadata-token-ttl-seconds: 21600"`
```

2) Use token to query metadata: 

```
root@ip-10-10-10-89:~# curl -H "X-aws-ec2-metadata-token: $TOKEN" -v http://169.254.169.254/latest/meta-data/iam/security-credentials/cg-banking-WAF-Role-cgidatyedvvzzy 
*   Trying 169.254.169.254...
* TCP_NODELAY set
* Connected to 169.254.169.254 (169.254.169.254) port 80 (#0)
> GET /latest/meta-data/iam/security-credentials/cg-banking-WAF-Role-cgidatyedvvzzy HTTP/1.1
> Host: 169.254.169.254
> User-Agent: curl/7.58.0
> Accept: */*
> X-aws-ec2-metadata-token: xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx==
> 
* HTTP 1.0, assume close after body
< HTTP/1.0 200 OK
< Accept-Ranges: bytes
< Content-Length: 1310
< Content-Type: text/plain
< Date: Mon, 20 Jan 2020 21:29:38 GMT
< Last-Modified: Mon, 20 Jan 2020 20:38:46 GMT
< X-Aws-Ec2-Metadata-Token-Ttl-Seconds: 21583
< Connection: close
< Server: EC2ws
< 
{
  "Code" : "Success",
  "LastUpdated" : "2020-01-20T20:38:30Z",
  "Type" : "AWS-HMAC",
  "AccessKeyId" : "xxxxxxxxxxxxxxxxxxxx",
  "SecretAccessKey" : "xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx",
  "Token" : "xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx==",
  "Expiration" : "2020-01-21T03:01:06Z"
* Closing connection 0
}root@ip-10-10-10-89:~# 
```

<HR>


## TEST 2:  update to support only IDMSv2

Note: by default, modify-instance-metadata-options command is not available on our instance.

ubuntu@ip-10-10-10-89:~$ aws ec2 modify-instance-metadata-options 
usage: aws [options] <command> <subcommand> [<subcommand> ...] [parameters]

The AWS CLI must be upgraded to aws-cli/1.16.287
Reference: https://blog.appsecco.com/getting-started-with-version-2-of-aws-ec2-instance-metadata-service-imdsv2-2ad03a1f3650

Our current version: 

```
ubuntu@ip-10-10-10-89:~$ aws --version 
aws-cli/1.14.44 Python/3.6.9 Linux/4.15.0-1057-aws botocore/1.8.48
```

Attempt to upgrade awscli through apt fails: as of Jan 25, 2020, this is the latest CLI version available for Ubuntu

```
ubuntu@ip-10-10-10-89:~$ sudo apt upgrade awscli
Reading package lists... Done
Building dependency tree       
Reading state information... Done
awscli is already the newest version (1.14.44-1ubuntu1).
Calculating upgrade... Done
0 upgraded, 0 newly installed, 0 to remove and 0 not upgraded.
```

<HR>

## Enable IMDSV2 (from an Amazon Linux host with role allowing access to EC2) 


View defaults before making changes: 

```
$ aws ec2 modify-instance-metadata-options --instance-id i-09c37ebd43840a574 
{
    "InstanceId": "i-09c37ebd43840a574", 
    "InstanceMetadataOptions": {
        "State": "pending", 
        "HttpEndpoint": "enabled", 
        "HttpTokens": "optional", 
        "HttpPutResponseHopLimit": 1
    }
}
```


Enable IMDSV2


```
$ aws ec2 modify-instance-metadata-options --instance-id i-09c37ebd43840a574 --http-tokens required --http-endpoint enabled
{
    "InstanceId": "i-09c37ebd43840a574", 
    "InstanceMetadataOptions": {
        "State": "pending", 
        "HttpEndpoint": "enabled", 
        "HttpTokens": "required", 
        "HttpPutResponseHopLimit": 1
    }
}
```

<HR> 


Test from remote machine:  With IMDSv2, we are no longer able to query the metadata for this instance!

```
root@HOSTNAME:~# curl http://18.214.25.228/latest/meta-data/iam/security-credentials/ -H 'Host:169.254.169.254'
<?xml version="1.0" encoding="iso-8859-1"?>
<!DOCTYPE html PUBLIC "-//W3C//DTD XHTML 1.0 Transitional//EN"
	"http://www.w3.org/TR/xhtml1/DTD/xhtml1-transitional.dtd">
<html xmlns="http://www.w3.org/1999/xhtml" xml:lang="en" lang="en">
 <head>
  <title>401 - Unauthorized</title>
 </head>
 <body>
  <h1>401 - Unauthorized</h1>
 </body>
</html>
```


<HR>

Change back to IMDSV1 (version 1)

```
[ec2-user@ip-10-10-20-29 ~]$ aws ec2 modify-instance-metadata-options --instance-id i-09c37ebd43840a574 --http-tokens optional --http-endpoint enabled
{
    "InstanceId": "i-09c37ebd43840a574", 
    "InstanceMetadataOptions": {
        "State": "pending", 
        "HttpEndpoint": "enabled", 
        "HttpTokens": "optional", 
        "HttpPutResponseHopLimit": 1
    }
}
```

Again the request to query metadata is successful:

```
ubuntu@ip-10-10-10-89:~$ curl http://169.254.169.254/latest/meta-data/iam/security-credentials/  -H 'Host:169.254.169.254'
cg-banking-WAF-Role-cgidatyedvvzzy
ubuntu@ip-10-10-10-89:~$ 
```



<HR>

## Additional Notes: 


Can we monitor logs to determine when someone has accessed the metadata service from outside the instance?

* Queries to metadata service (link local address) are not captured in flow logs:
* https://docs.aws.amazon.com/vpc/latest/userguide/flow-logs.html 
     
* With the announcement of IDMSV2 “CloudTrail” is also being updated to record the new ec2:RoleDelivery parameters. 
* https://aws.amazon.com/blogs/security/defense-in-depth-open-firewalls-reverse-proxies-ssrf-vulnerabilities-ec2-instance-metadata-service/
  
Vulnerability scanning:   

* Scanning our vulnerable instance wtih AWS Inspector did not result in any related medium or high severity findings. 
(Security Groups are configured so that the vulnerable instance is actually only visible from a single IP.)
  
