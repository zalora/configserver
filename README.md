# Spring Cloud Config Server on AWS ElasticBeanstalk

## Requirements

- Access the git repo via ssh
- Passwords, keys and other secrets must not be committed to the repo
- Sensitive data must be stored encrypted in the repository
- Setup must be HA, because other services depend on it

## Used AWS Services

- AWS CodeCommit to store the code
- AWS ElasticBeanstalk

## How to configure everything

Copy or rename the example ElasticBeanstalk config file:

`$ cp .ebextensions.config.example .ebextensions.config`

Replace the content keys in the newly created file. I base64-encoded some of the content to make 
sure the whitespaces are conserved properly. You can base64 files and strings from the command line like this:

File: `$ base64 -w 0 <file>`

String: `$ echo "Hello Config Server" | base64 -w 0`

### predeploy.config

The file contains 4 sections:

#### config

```
host git-codecommit.*.amazonaws.com
    User APXXXXX6CXYYYIQXXXXX
```

Open IAM in your AWS console, create a user and assign it an SSH-key. Use the SSH key ID as the username. Make sure your
AWS user has to right permissions to access the repo. You can do that by assigning a policy:

```
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": [
                "codecommit:BatchGetRepositories",
                "codecommit:Get*",
                "codecommit:GitPull",
                "codecommit:List*"
            ],
            "Resource": "<arn_of_your_codecommit_repo>"
        }
    ]
}
```

#### id_rsa + id_rsa.pub

Add the key you uploaded to for your IAM user before and add it in the corresponsing places in the config file.

#### known_hosts

You can either disable the host key checks, which is absolutely not recommended or add the Amazons's CodeCommit host to
the `known_hosts` file. Amazon publishes their fingerprints [here](http://docs.aws.amazon.com/codecommit/latest/userguide/setting-up-ssh-unixes.html), 
so you can compare if they match:

``ssh-keygen -lf /dev/stdin  <<< `ssh-keyscan -t rsa git-codecommit.us-west-2.amazonaws.com` ``

If everything's ok, fetch the key with `$ ssh-keyscan -t rsa git-codecommit.us-west-2.amazonaws.com` and add it to the
known_hosts content section in the config file.
  
### src/main/resources/bootstrap.yml

Copy the example bootstrap file and update the url to your repository. I prefer to set the encryption key via an ENV
variable, but it's also possible to set it in the config file.

## Deploy

After configuring everything a `$ mvn clean package` creates a so called deploy package in the target folder, which is
a zip file containing everything EB needs to get going:

- The fat jar 
- The Procfile, which tells EB how and which file to run
- The .ebextensions folder, which contains the deploy instructions for Beanstalk
