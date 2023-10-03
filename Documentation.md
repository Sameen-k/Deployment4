# Deployment 4
## Purpose
The purpose of this deployment is to understand the uses of CloudWatch by pushing the limits of a T2 medium through usage of nginx. By running the Jenkins build, we can observe the EC2's CPU utilization. We can use our observations to compare to a T2 micro and infer the results if this deployment was run on one.

## System Diagram

![Deployment4 drawio (2)](https://github.com/Sameen-k/Deployment4/assets/128739962/265c9f7d-0497-4220-8276-9778345b0ec2)

## Steps

### EC2 Instance Configurations
The first thing to make sure is that the ec2 instance is properly configured. Before creating it, it's important to make sure that VPC is created for the EC2 first as well as the necessary security group configurations have been set as so:
The VPC should have 2 public and 2 private subnets.

![Screen Shot 2023-09-28 at 7 09 11 PM](https://github.com/Sameen-k/Deployment4/assets/128739962/f959b6b0-c30a-4dd5-9587-619cc4ed097a)
*Security groups should include ports 80, 8080, 8000, and 22*

The AWS routing table map feature can help with the organization which should look like the following:

![Screen Shot 2023-09-28 at 6 59 17 PM](https://github.com/Sameen-k/Deployment4/assets/128739962/351b8174-ccf3-4252-96d7-38dd9dd30862)

Now when creating the EC2 the customizations previously made are selected and the T2 Medium option is selected to be created in the public subnet A

### Installations on The T2 Medium
After the instance has launched, make sure to first update the instance before making the following installations, use the command:

``sudo apt update``

Now you're ready to make the next installations to be able to run the application. First, install the following version of Python, use the command:

``sudo apt install python3.10-venv``

Then install python-pip by using the following command:

``sudo apt install python3-pip``

Then install NGINX with the following command 

``sudo apt install nginx``

At this point, you may also install Jenkins but do not run any builds!

### Git Commands
After the installations, you may now clone, branch, edit, merge, and push the commit.
First start with a git clone along with a link to the repository you are cloning from:

``git clone <repolink>``

Then make sure you ``cd`` into the folder that is made so you can do the next few commands! Next, you can create a branch where you will make the necessary changes by using the following command:

``git branch <branchname>`` 

Then checkout the branch

``git checkout <branchname>``

Then you can ``sudo nano`` into the Jenkinsfile where you can make a necessary change.
The above should all look like the following Image:

![Screen Shot 2023-10-02 at 2 05 14 PM](https://github.com/Sameen-k/Deployment4/assets/128739962/a907fee8-02e2-48a4-aede-689cfc0c6024)

Now that you're in the Jenkinsfile, you can completely erase the existing code and replace it with the following:

```
pipeline {
agent any
stages {
stage ('Build') {
steps {
sh '''#!/bin/bash
python3 -m venv test3
source test3/bin/activate
pip install pip --upgrade
pip install -r requirements.txt
export FLASK_APP=application
'''
}
}
stage ('test') {
steps {
sh '''#!/bin/bash
source test3/bin/activate
py.test --verbose --junit-xml test-reports/results.xml
'''
}
post{
always {
junit 'test-reports/results.xml'
}
}
}
stage ('Clean') {
steps {
sh '''#!/bin/bash
if [[ $(ps aux | grep -i "gunicorn" | tr -s " " | head -n 1 | cut -d " " -f 2) != 0 ]]
then
ps aux | grep -i "gunicorn" | tr -s " " | head -n 1 | cut -d " " -f 2 > pid.txt
kill $(cat pid.txt)
exit 0
fi
'''
}
}
stage ('Deploy') {
steps {
keepRunning {
sh '''#!/bin/bash
pip install -r requirements.txt
pip install gunicorn
python3 -m gunicorn -w 4 application:app -b 0.0.0.0 --daemon
'''
}
}
}
}
}
```
After making this change to the Jenkins file, you now have to commit these changes, merge the branches, and push. 
run the following command to commit the changes:

``git commit -m "Jenkins file update"``

Then run the merge command. First return to the main branch: ``git checkout main`` After this, you can merge the branch by using the following command: ``git merge <branchname>``
You can push the changes with the following command: ``git push``
After running the git push, you will be prompted to enter your Git username and password. Make sure, for the password to use a personal access token that you can generate on GitHub.

<img width="1350" alt="Screen Shot 2023-10-02 at 5 22 35 PM" src="https://github.com/Sameen-k/Deployment4/assets/128739962/4fa27449-e01d-4c2c-9c8e-befaaf53c854">

### Configuring NGINX
After you've pushed your changes to the Jenkins file. Now you need to configure NGINX. For this step, you just have to edit the configuration file following this path: ``/etc/nginx/sites-enabled/default``
Once in the file, you can make the following changes:

```
###First change the port from 80 to 5000, see below:
server {
listen 5000 default_server;
listen [::]:5000 default_server;

####Now scroll down to where you see “location” and replace it
with the text below:

location / {
proxy_pass http://127.0.0.1:8000;
proxy_set_header Host $host;
proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
}

```
Make sure you save the file before exiting!

### CloudWatch Configuration
We can now configure the CloudWatch agent before running the build. To do this, you can head to your AWS account and first create the role for CloudWatch agent. 
after selecting **Create Role**
Keep AWS Service selected, then for the use case select EC2, then search for CoudWatchAgent, then select CLoudWatchAgentAdminPolicy.
After creating the role you must attach the role to the EC2 Instance by doing the following: actions/security/modify IAM role, then select the role you just created and then select **update IAM role**

Now you can go to your EC2 Instance terminal and download the agent.
first, make sure to update your EC2:

``sudo apt update``

Then run the following command:

``wget https://amazoncloudwatch-agent.s3.amazonaws.com/ubuntu/amd64/latest/amazon-cloudwatch-agent.deb``

Then run this next command:

``sudo dpkg -i -E ./amazon-cloudwatch-agent.deb``

Then to configure the agent you can run this command so you can customize what you want to monitor:

``sudo /opt/aws/amazon-cloudwatch-agent/bin/amazon-cloudwatch-agent-config-wizard``

You can monitor whatever you like but make sure you include CPU and Memory at the very least for this deployment

Then run this command after configuring to save your settings:

``sudo /opt/aws/amazon-cloudwatch-agent/bin/amazon-cloudwatch-agent-ctl -a fetch-config -m ec2 -s -c file:/opt/aws/amazon-cloudwatch-agent/bin/config.json``

### Jenkins Build

You can finally run your Jenkins build! After going to Jenkins, you can select Multi-branch Pipeline. Make sure to add the plugin: “Pipeline Keep Running Step”
Then you can run your build which should look like this:

<img width="1351" alt="Screen Shot 2023-10-02 at 6 09 31 PM" src="https://github.com/Sameen-k/Deployment4/assets/128739962/9d44f363-2acb-4973-ae58-5efc134e42bc">

### CloudWatch Observations:

<img width="1365" alt="Screen Shot 2023-10-02 at 5 41 50 PM" src="https://github.com/Sameen-k/Deployment4/assets/128739962/b5d02bf7-a690-4934-8a99-0c8c729a987e">

After looking at this general view of the metrics for the server, we can observe that the CPU usage is very high. And the application itself is taking a while to load.

After running a second Jenkins build, we can observe a spike in CPU usage in CloudWatch

<img width="1117" alt="Screen Shot 2023-10-02 at 10 40 37 PM" src="https://github.com/Sameen-k/Deployment4/assets/128739962/08d617e8-e301-4b38-a5d7-416168d0f342">

With this observation, we can understand that if this exact application were to be run the same way on a T2 micro instance, it would fail to work as the amount of CPU available is greatly reduced in a T2 micro compared to a T2 Medium. As we can see, the T2 Medium is already very close to using maximum of the CPU it has. 

## Optimization

The best way to improve this build would be to use a larger T2 instance size so it handles the deployment better or continue using Elastic Beanstalk to run the application. 
