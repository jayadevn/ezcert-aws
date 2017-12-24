# ezcert-aws

This tool is an easy way for users of AWS to generate certs with LetsEncrypt.

## How it works

When you launch the CloudFormation template in this project, you will get an EC2 instance with the LetsEncrypt certbot installed and a script that's ready for you to run to generate your certs.  You will then SSH to this instance and run the script, following some self-explanatory prompts.  At the end of this process, your cert and private key are uploaded to a KMS-encrypted S3 bucket you provided.

## Why I created this

For my own personal projects, I was finding that I was doing a ton of fiddling around and rereading the internet every time I had to get an SSL cert.  LetsEncrypt has a great tool for obtaining certificates called certbot.  A lot of the instructions out there assume that you're configuring the SSL cert on one of a number of popular platforms (e.g. Apache), on the host on which the service will actually run, which wasn't true for me.  This tool takes advantage of certbot's standalone mode, which just gets the certificate, and automates the fiddling you need to do around that.

## Instructions

Overview of steps:
1. Collect some AWS details you'll need to start.
1. Launch the CloudFormation template in this project
1. SSH to the EC2 instance it creates
1. Run a script on that EC2 instance that will check your S3/KMS settings and permissions; generate a certificate; and upload the materials to your S3 bucket.
1. Delete the CloudFormation stack.

### Before starting

Have these ready:
* A Route53 public hosted zone on a domain you own, e.g. example.com.
* The desired commonName for the certificate, e.g. certifyme.example.com
* An S3 bucket, with KMS encryption enabled, to which the keys will be delivered
* A VPC, public subnet, SSH key, and a CIDR range from which you will SSH

#### Securing the private key

The private key associated with the cert you will generate is a secret, so you need to guard it carefully.  Don't lose control of it.  To help you guard your private key, the S3 bucket to which it will be uploaded is required to be KMS-encrypted; thus, even if you mess up the S3 bucket permissions, the data in there will be inaccessible unless they also have access to its KMS key.  

As a result, you should set the permissions on the KMS key are as restrictive as you can tolerate.  What the AWS IAM console does for you is not as restrictive as it gets but is good enough if you trust everyone who has access to your AWS account.  To learn more: This is a pretty detailed rundown of how to do permissions in KMS http://docs.aws.amazon.com/kms/latest/developerguide/key-policies.html#key-policy-default.  Note that whatever permissions you give to the ...:root principal mean that any user in the account with appropriate policy on that user can do those actions on the key.  If you need to be very sure that Decrypt operations are possible only by specific IAM principals in the account ever, then do not give this permission to the root principal.

### Launch the CloudFormation template

Go to Cloudformation in the AWS console to launch the template at ezcert.yaml in this project.

Please note that if the S3 bucket you specify does not exist or is not KMS-encrypted, you will hit a failure about that later in this process.

### SSH to the EC2 instance

The EC2 instance will be launched at the DNS name for which you're issuing the cert.  (We did this as a way of proving to LetsEncrypt that we do indeed control this DNS record.)  To connect:

```bash
ssh -i /path/to/sshkey.pem ubuntu@certifyme.example.com
```

### Run the script

Before proceeding, you'll want to make sure the cloud-init script has run.  When you see a file /home/ubuntu/ezcert.rb, you're ready.  It takes about four minutes.

You will need to run the ezcert script as root because certbot requires root privilege.

```bash
ubuntu@ip-172-30-2-115:~$ sudo ./ezcert.rb
```

This script will first test that it has the right access to S3 and KMS, and then it will have certbot generate a cert.  The certs will appear at /etc/letsencrypt/live/certifyme.example.com, and they will be uploaded and automatically encrypted at your S3 bucket.  

### Delete the CloudFormation stack

Once you have your certificates, you can delete the CloudFormation stack; you no longer need any of the resources it created.

# Revoking certs

You can use this tool to revoke certificates as well, regardless of whether you issued them using the above method or not.  This method assumes you've lost the private key; if you still have the private key, then it's even easier.

Overview of steps:
1. Go to https://cert.sh, look up your certificate, and download the PEM.
1. Launch this CloudFormation template
1. From the EC2 instance, use certbot to request a certificate for both the name you're trying to revoke (e.g. certifyme@example.com) and for a name that doesn't exist (somethingelse-doesntexist@example.com.  The point of this step is to prove to LetsEncrypt that you control the DNS record for certifyme@example.com
1. From the EC2 instance, use certbot to revoke the certificate you downloaded.
1. Delete the CloudFormation stack

In detail:

You can look up the certificate you're trying to revoke by searching at https://cert.sh.  Search for the name of the certificate, e.g. certifyme@example.com.  Go to the detail page for your certificate, and click the "Download PEM" link to download the certificate.  It will have a name like 123456789.crt.

Then launch this CloudFormation template and SSH to the EC2 instance it launches.

Get the certificate onto your EC2 instance, e.g. at /path/to/123456789.crt.

Then run these two commands.

```bash
# When you run this, you will see an error telling you that it couldn't connect to nope-on-a-rope.example.com.  This is expected.  You're doing this step to get LetsEncrypt to put some credentials on your host; you don't actually want a certificate out of this.
ubuntu@ip-172-30-2-100:~$ sudo certbot certonly -n --standalone --preferred-challenges http --http-01-port 80 -d certifyme.example.com -d nope-on-a-rope.example.com --email foo@bar.com --agree-tos
```

```bash
# You'll see an error message for this one as well, like "No match found for cert-path /path/to/123456789.crt.  That's because you don't actually have this cert on this machine.  The revocation succeeded, and you can check that at https://crt.sh; there is a a "Check" link in the OCSP row in your certificate details there.
ubuntu@ip-172-30-2-100:~$ sudo certbot revoke -n --cert-path /path/to/123456789.crt
```
