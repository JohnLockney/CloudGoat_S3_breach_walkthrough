## Test 1: Restrict access to metadata service thruogh iptables

<HR> 

1) SETUP 
 
Get SSH accsess to instance 
    dismount EBS volume
    mount volume on alternative EC2 (in the same AZ!!!)
    copy authorized_keys from new instance to mounted volume
    

<HR>

2) Confirm instance role has access to list S3 buckets:

```
root@ip-10-10-10-89:~# aws s3 ls
2020-01-12 19:07:01 cg-cardholder-data-bucket-cgidatyedvvzzy
root@ip-10-10-10-89:~# 
```  
 

Verify instance can access metadata:

 ==============================================================================
 

3) Get Current role:


```
  root@ip-10-10-10-89:~# aws sts get-caller-identity
{
    "UserId": "XXXXXXXXXXXXXXXXXXXXX:i-09c37ebd43840a574",
    "Account": "xxxxxxxxxxxxxxx",
    "Arn": "arn:aws:sts::xxxxxxxxxx:assumed-role/cg-banking-WAF-Role-cgidatyedvvzzy/i-09c37ebd43840a574"
}
```


4) Edit instance role, remove policy allowing S3 full access



5) Check S3 access from instance:
    
```
root@ip-10-10-10-89:~# aws s3 ls
An error occurred (AccessDenied) when calling the ListBuckets operation: Access Denied
```

<HR>

6) Update iptables to prevent instance from accessing metadata:

```   
List default rules:

    root@ip-10-10-10-89:~# iptables -L
    Chain INPUT (policy ACCEPT)
    target     prot opt source               destination         

    Chain FORWARD (policy ACCEPT)
    target     prot opt source               destination         

    Chain OUTPUT (policy ACCEPT)
    target     prot opt source               destination         
    root@ip-10-10-10-89:~# 
```


Add rule to prevent access to metadata:

```
    root@ip-10-10-10-89:~# iptables -A OUTPUT  -d 169.254.169.254 -j DROP
```

List updated rules:

```
    root@ip-10-10-10-89:~# iptables -L  
    Chain INPUT (policy ACCEPT)
    target     prot opt source               destination         

    Chain FORWARD (policy ACCEPT)
    target     prot opt source               destination         

    Chain OUTPUT (policy ACCEPT)
    target     prot opt source               destination         
    DROP       all  --  anywhere             169.254.169.254     
```


With the new iptables rule, the instance can no longer query metadata: 

```
    root@ip-10-10-10-89:~# curl http://169.254.169.254/latest/metadata
    ^C

    root@ip-10-10-10-89:~# aws sts get-caller-identity
    Unable to locate credentials. You can configure credentials by running "aws configure".

    root@ip-10-10-10-89:~# aws s3 ls
    Unable to locate credentials. You can configure credentials by running "aws configure".
```

<HR> 

7) Flush iptables (to remove rules)

```
    root@ip-10-10-10-89:~# iptables -F OUTPUT

    root@ip-10-10-10-89:~# iptables -L
    Chain INPUT (policy ACCEPT)
    target     prot opt source               destination            
    Chain FORWARD (policy ACCEPT)
    target     prot opt source               destination            
    Chain OUTPUT (policy ACCEPT)
    target     prot opt source               destination
```
    
Check current credentials:
    

```
root@ip-10-10-10-89:~# aws sts get-caller-identity
{
    "UserId": "AROA6KQH3SZZWM2UD3WXR:i-09c37ebd43840a574",
    "Account": "xxxxxxxxxxxxxxxx",
    "Arn": "arn:aws:sts::xxxxxxxxxx:assumed-role/cg-banking-WAF-Role-cgidatyedvvzzy/i-09c37ebd43840a574"
}
```         

8)  Re-attach s3 full access policy to role:

```
    root@ip-10-10-10-89:~# aws s3 ls
    2020-01-12 19:07:01 cg-cardholder-data-bucket-cgidatyedvvzzy
    2019-06-30 23:54:06 magic-website-stuff.net
```

<HR>

Test 2:  Deny access to metadata except for ‘root’ user

```
    root@ip-10-10-10-89:~# iptables -A OUTPUT -m owner ! --uid-owner root -d 169.254.169.254 -j DROP
```

Review current rules: 

```
    root@ip-10-10-10-89:~# iptables -L
    Chain INPUT (policy ACCEPT)
    target     prot opt source               destination         

    Chain FORWARD (policy ACCEPT)
    target     prot opt source               destination         

    Chain OUTPUT (policy ACCEPT)
    target     prot opt source               destination         
    DROP       all  --  anywhere             169.254.169.254      ! owner UID match root
    root@ip-10-10-10-89:~# 
```


Confirm ‘root’ can access metadata:

```
        root@ip-10-10-10-89:~# aws sts get-caller-identity
            {
            "UserId": “xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx”,
            "Account": "XXXXXXXXXXXXXXXXx",
            "Arn": "arn:aws:sts::xxxxxxxxxxx:assumed-role/cg-banking-WAF-Role-cgidatyedvvzzy/i-09c37ebd43840a574"
            }
```            


Confirm non-root access reqeust for metadata access is denied: 

```
        ubuntu@ip-10-10-10-89:~$ aws sts get-caller-identity    
        Unable to locate credentials. You can configure credentials by running "aws configure".
        ubuntu@ip-10-10-10-89:~$ 
```

Is the instance able to query metadata for role changes to take affect? 
        
        
```        
        root@ip-10-10-10-89:~# aws s3 ls
        An error occurred (AccessDenied) when calling the ListBuckets operation: Access Denied
```        

Re-attach S3 full access policy to role, root user is again able to query S3
       
              
<HR>


## Test with iptables: 

       
Test 1: (No iptables rules)

(REMOTE HOST)

```
root@HOSTNAME:~# curl -s http://100.25.77.36/latest/meta-data/iam/security-credentials/ -H 'Host:169.254.169.254'  
cg-banking-WAF-Role-cgidatyedvvzzy
root@HOSTNAME:~# 
```

Test 2: (only root may access metadata)

```
# iptables -A OUTPUT -m owner ! --uid-owner root -d 169.254.169.254 -j DROP
#
```


```
root@HOSTNAME:~# curl -s http://100.25.77.36/latest/meta-data/iam/security-credentials/ -H 'Host:169.254.169.254'  
<html>
<head><title>504 Gateway Time-out</title></head>
<body bgcolor="white">
<center><h1>504 Gateway Time-out</h1></center>
<hr><center>nginx/1.14.0 (Ubuntu)</center>
</body>
</html>
root@HOSTNAME:~#
```

Note: This prevents remote access through proxy service. (Unless the proxy service runs as root?)
