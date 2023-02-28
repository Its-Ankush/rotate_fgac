# FGAC password rotation

Here is the idea - 

* The entire thing will run on lambda because cron on an EC2 will be old school and lambda eliminates the maintenance of servers [EC2]
* Lambda will need - SNS permission, Secretsmanager for creating a random password, AWSLambdaBasicExecutionRole for logging into CWL and AmazonOpenSearchServiceFullAccess for changing the master password
* First lambda will generate a strong password with Secrets Manager.
* It will then encrypt it with the help of a KMS key.
* Then an SNS notification will be send to the subscriber [here I've used Email] and the body will contain the encrypted password.
* It is also important to log the encrypted password in CWL so a simple print statement will also do. Just in case the email gets deleted so it will still be there in CWL logs
* Lambda will actually change the password of master user with the original unencrypted password that you generated with Secrets Manager
* Log the response with just a print statement
* Push that response to SNS and to the email
* Lastly this entire setup will run at regular intervals based on an EventBridge trigger for full automation

### Creating the policies

RandomPasswordGen

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "VisualEditor0",
            "Effect": "Allow",
            "Action": "secretsmanager:GetRandomPassword",
            "Resource": "*"
        }
    ]
}
```

**AWSLambdaBasicExecutionRole** and **AmazonSNSFullAccess** and **AmazonOpenSearchServiceFullAccess**. If you are using a VPC based domain, then you will also have to use **AWSLambdaVPCAccessExecutionRole** policy and attach it to the below mentioned role 

Now attach these policies to a role and name it something like "LambdaRoleForPasswordAutomation"

Since this policy is for lambda, here will be the trust policy

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Principal": {
                "Service": "lambda.amazonaws.com"
            },
            "Action": "sts:AssumeRole"
        }
    ]
}
```

Create a normal symmetric kms key. Just make sure to select "**LambdaRoleForPasswordAutomation**" for both in k**ey administrators** and also in **key usage permission**

Please also note that you have to select **LambdaRoleForPasswordAutomation** as the lambda execution role and also make sure you select the timeout as 1 min and DON'T keep the default timeout since it will timeout because there are a lot of steps involved.

## The lambda code

Make sure to create an SNS topic and add email subscribers and verify the email

Here the runtime is Python 3.9 for lambda


```python
import json
import base64
import boto3
# This will create a password based on password length with the help of secrets manager
class SecretsManagerSecret:
    def __init__(self, secretsmanager_client):
        self.secretsmanager_client = secretsmanager_client
        self.name = None

    def get_random_password(self, pw_length):
        response = self.secretsmanager_client.get_random_password(PasswordLength=pw_length)
        password = response['RandomPassword']
        return password

# USE KMS to encrypt password coz obviously plaintext passwords are a bad idea. The return type is base64 encoded string.The encrypted password will be sent to CWL and EMAIL :)

class KeyEncrypt:
    def __init__(self, kms_client):
        self.kms_client = kms_client


    def encrypt(self, key_id,text):
        blob = self.kms_client.encrypt(
        KeyId=key_id, Plaintext=text)['CiphertextBlob']
        b64blob=base64.b64encode(blob).decode()
        return b64blob


def lambda_handler(event, context):
# Generate password of 10 characters and encrypt it using KMS the "keyid" is your KMS keyid 
  secret = SecretsManagerSecret(boto3.client('secretsmanager'))
  password=secret.get_random_password(10)
  kms=KeyEncrypt(boto3.client('kms'))
  keyid="984***7bb4";
  cipher=kms.encrypt(keyid,password);
# Print to keep a log of encrypted password in CWL
  print (f"The encrypted password is {cipher}")
# Send this info to SNS and from SNS to email
  client = boto3.client("sns")
  response = client.publish(
    TopicArn='<sns_arn_without_angular_brackets>',
    Message=json.dumps({
    "Password will be changed to":cipher
    })
  )
  
# Actually change the password to the randomly generated password from secrets manager. Here I have hardcoded the username as minhal. Minhal is my buddy
  
  client = boto3.client('opensearch');
  response2 = client.update_domain_config(
        DomainName='saml',
        AdvancedSecurityOptions={
            'MasterUserOptions': {
                'MasterUserName': 'minhal',
                'MasterUserPassword': password
        }   
        }   
        )

# Print statement to log the response to Cloudwatch logs

  print (response2)

# Share the response to SNS and then to email
  client = boto3.client("sns")
  response3 = client.publish(
    TopicArn='<sns_arn_without_angular_brackets>',
    Message=json.dumps({
    "Here is the output after changing the password":str(response2)
    })
  )

        
  return {
    'body': "Finish executing script :)"
  }
```


<img width="1326" alt="kmsid" src="https://user-images.githubusercontent.com/68053631/221889702-f844df93-0713-4a77-ba3a-640129d5ce91.png">


Please go through the code once. The code is pretty self explanatory as I have commented every steps that I followed.

### Points to note
* Here I have used Email as SNS subscribers.
* Before the password actually changes, you will receive an email with the encrypted password.
* After the password change snippet is run, you will receive an output of the same in your mail
* You can select an Event Bridge trigger like this - and this will run after every 5 mins

<img width="640" alt="trigger" src="https://user-images.githubusercontent.com/68053631/221877910-7da50003-0188-492c-93a4-795b4a7eebe7.png">



## How to decrypt the cipher text password ?

Thats pretty easy too. Before that keep in mind that only certain people who are allowed in the KMS key access policy will be able to work with the key. The cx would need to allow principals or roles in order for them to use this key. This will also bring in role separation which might be desired/useful.



Here is how you will decrypt the cipher - 

```python
import base64
import boto3
class KeyEncrypt:
    def __init__(self, kms_client):
        self.kms_client = kms_client
    def decrypt(self, key_id, cipher_text):
        text = self.kms_client.decrypt(
        KeyId=key_id, CiphertextBlob=base64.b64decode(cipher_text))['Plaintext']
        return text.decode()


kms=KeyEncrypt(boto3.client('kms'))
keyid="984***7bb4";
cipher="AQICAHiB1knvodCPBjroelVgpKw29+X+lI1Nu3mhPWhEivP2vgERC3CTnbzenZRblrBTyxn0AAAAaDBmBgkqhkiG9w0BBwagWTBXAgEAMFIGCSqGSIb3DQEHATAeBglghkgBZQMEAS4wEQQMUO1XWAA6XszoA8UlAgEQgCW2SElG/5Mc2y8KtlGjBTmTrWZHV88y7rACCjLZptVQ5ZVQbyn2"
decipher=kms.decrypt(keyid,cipher);
print (decipher)
#output "the earth is flat"
```

You will get the cipher as an email and also in cloudwatch logs. Just copy and paste that into the "**cipher**" variable in python code or you can also use the cli as well. Here I'm using boto3 to make things simple so that I can run this in lambda as well without using my custom runtime which is a pain.


## Drawbacks of this code

This might not be production ready and may contain bugs. This is tested by me on my test domain. You can also see that I have not added any exception statements as well. Again this code is one giant monolith which could have been more distributed and made much more elegant. If you wish to use this, please do remember to encrypt credentials before logging them or sending them via email.

#### Credits - me :)
