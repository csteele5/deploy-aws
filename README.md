# deploy-aws

This repository, stack and application are the end result of a tutorial for EC2, Docker, RDS + Cloudformation. Tutorial is a YouTube video currently here: https://www.youtube.com/watch?v=8J0g_xWUzV0

# Methodology Outline

- Create AWS KeyPair for SSH access to EC2 instance
- Configure AWS CLI
- Define stack in YAML - containing EC2 instance
- Update stack definition to install docker on EC2 instance
- Update stack to add RDS for MySQL database
- Define Docker Compose for application deployment
- Deploy the application using docker compose & test

# Build Process

Follow the steps outlined in the above video.  See notes below for more information.

# Build Notes

Create new KeyPair from EC2 dashboard and copy into new project folder
- change properties, ie: chmod 400 aws-key1.pem
(SSH will reject the key if it is publicly visible)

Create a new CF user in IAM with both console and cli access
Add to CLI - aws configure
**** TIP FOR SPECIFIC REPO.  Create a profile
aws configure --profile demo  (I missed this first time through, so none of the below commands have the profile param associated with them!)

Create new stack in CF - This guy builds a stack.yml from scratch.  Name does not matter.
Then tests it from CLI
aws cloudformation create-stack --stack-name blog-stage --template-body file://$PWD/stack.yml --profile demo --region us-west-2 (THIS DID NOT WORK.  It couldn't resolve PWD and the profile param was extraneous)

aws cloudformation create-stack --stack-name blog-stage --template-body file://stack.yml --region us-west-2 (This worked)

To show created resources in CLI:
 aws cloudformation describe-stack-resources --stack-name blog-stage

SSH into new instance:
ssh -i aws-key1.pem ec2-54-188-43-146.us-west-2.compute.amazonaws.com
CRAP: cwsteele@ec2-54-188-43-146.us-west-2.compute.amazonaws.com: Permission denied
https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/TroubleshootingInstancesConnecting.html
SAYS DO THIS: 
ssh -i aws-key1.pem ubuntu@ec2-54-188-43-146.us-west-2.compute.amazonaws.com   YES!!!

NOW ADD DOCKER and Stuff.
Shut down/Delete first
aws cloudformation delete-stack --stack-name blog-stage

recreate stack, ssh in and check on docker:  ps -ef | grep docker
which docker ?

The video example of bootstrapping docker is old:
CHECK HERE FOR A SNIPPET: https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/quickref-ecs.html
THIS WORKS:
UserData: !Base64 |
        #!/bin/bash
        apt-get update
        apt-get install -y apt-transport-https ca-certificates curl software-properties-common
        curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -
        add-apt-repository \
           "deb [arch=amd64] https://download.docker.com/linux/ubuntu \
           $(lsb_release -cs) \
           stable"
        apt-get update
        apt-get install -y docker-ce
        usermod -aG docker ubuntu

MORE INFO ON DOCKER SETUP: ec2 should not be micro, more HD space
https://medium.com/@cjus/installing-docker-ce-on-an-aws-ec2-instance-running-ubuntu-16-04-f42fe7e80869

NEXT STEP:  alter docker command and add port to allow management of docker externally
From this page:
https://docs.docker.com/config/daemon/systemd/
Add these lines:
mkdir -p /etc/systemd/system/docker.service.d
        printf "[Service]\nExecStart=\nExecStart=/usr/bin/dockerd -H fd:// -H tcp://0.0.0.0:2375\n" >>  /etc/systemd/system/docker.service.d/docker.conf
        systemctl daemon-reload
        systemctl restart docker
THEN add new Ingress for port 2375

AFTER recreating stack, verify that docker is listening on 2375
docker -H tcp://18.237.85.207:2375 ps -a
DOCKER SETUP COMPLETE - MO FO!

NOW, add database stuff - mysql
and the application stuff comes from here:
https://github.com/devteds/e6-deploy-on-aws   (more good stuff here)
Copy the staging.yml to app.yml in my repo, replace the DB Host endpoint with the name after the build

Trick to check docker:
INSTEAD OF docker -H tcp://18.237.85.207:2375 ps -a
DO this:
export DOCKER_HOST=tcp://18.237.85.207:2375
then: docker ps -a   will work

THE STACK IS NOW READY TO DEPLOY THE APPLICATION
From the repo directory - migrate the database from the tutorial repo:
docker-compose -f app.yml run --rm app rails db:migrate
NEXT:  docker-compose -f app.yml up -d
Verify:  docker-compose -f app.yml ps

The API should now be accessible via the public IP or DNS.
Postman - GET to ec2-54-186-91-137.us-west-2.compute.amazonaws.com/posts
(nothing comes back)
Populate some test data
docker-compose -f app.yml run --rm app rails db:seed

Now run the above postman again for list, then run this for specific record
ec2-54-186-91-137.us-west-2.compute.amazonaws.com/posts/1.json

THERE ARE SECURITY HOLES and other issues to review from end of tutorial.
Other than that, this is complete!

