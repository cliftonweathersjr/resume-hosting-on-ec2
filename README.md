# Hosting Resume on AWS EC2 with a CI/CD pipeline using GitHub Actions

In this project, I hosted my resume on an AWS EC2 instance with a Continuous Integration/Continuous Deployment pipeline setup using GitHub Actions.

## Step One: Create an EC2 Instance and Download Key Pair

To start, locate the EC2 dashboard under services and click `Launch Instance` to begin customizing your instance settings. For this instance, an Ubuntu LTS image, 64-bit (X86) architecture and a t2.micro instance type will suffice.

Afterwards, a SSH/Key Pair will need to be created. The settings can be left as default and only the name will need to be given. After selecting `Create key pair`, the file will be downloaded to your machine. The location will need to be noted, because it will be used later.

Final setting that needs to be addressed before launching the instance is the Network Settings. For the instance, a security group will need to be created. For this security group, SSH, HTTPS, and HTTP traffic is going to be allowed from Anywhere (0.0.0.0/0). This will allow successfull connections to the instance from browsers to view the resume.

The EC2 instance can now be launched and after a few minutes, the state of the instance should show as running and under "status check" all checks should pass.


## Step Two: Create GitHub Repository and Secrets

To automate the process of updating the resume website, GitHub is going to be used to establish a CI/CD pipeline. After logging into github.com, a repository will need to be made to hold the necessary files. This can be accomplished by selecting the green 'New' button under the respository section. After customizing the name, description and privacy status, the files can be uploaded. This should include the resume in HTML format and any CSS files, if needed.

After uploading files, Secrets will need to made in the GitHub repository. These are used as environment variables in the GitHub Action to gain access to the EC2 instance.

To begin creating the Secrets, go to the settings for the respository. Under the dropdown menu for `Secrets and Variables` in the Security section, click on Actions. From there, you are able to click the `New Repository Secret` button. The four following secrets will need to be created.

+ EC2_SSH_KEY: This will be the contents of the .pem file created earlier with the EC2 instance.
+ HOST_DNS: Public DNS record of the instance. This can be found under the instance details.
+ USERNAME: `Ubuntu` for Ubuntu Images, `ec2-user` for Amazon Linux 2 AMI images.
+ TARGET_DIR: This is where the code is deployed. For this project, `Home` will be the directory.


## Step Three: Creating Workflow for GitHub Actions

A workflow is essentially your CI/CD setup for GitHub Actions. To do this, a YAML file will need to be created. This can be done by selecting `Add File` and then choosing `Create new file`. A `.github/workflows` directory will need to be created in the repository if it does'nt already exist. In that folder, the YAMl fill be named, `github-actions-ec2.yml`.

The first part of our YAML will be used to define the trigger for the continuous deployment. The code below will set a name and ensure that your Actions only run when you push to he main branch.
```
name: Push-to-EC2

# Trigger deployment only on push to main branch
on:
  push:
    branches:
      - main
```

The second part of the file will start defining the jobs. The jobs are steps that you can define and see individual status reports on when checking the logs in the Actions section.
```
jobs:
  deploy:
    name: Deploy to EC2 on main branch push
    runs-on: ubuntu-latest
```

After defining the jobs, it will need to checkout the pushed code to the runner by a GitHub defined action named `actions/checkout@v4`. That code will look like such.
```
steps:
      - name: Checkout the files
        uses: actions/checkout@v4
```

The final steps of the file will be deploying the code to the server and running the necessary commands. To do this, the runner will need to access the EC2 using SSH and perform rsync. This will be accomplished using another GitHub action, `easingthemes/ssh-deploy`. After that, remote ssh commands will needed to be executed using ssh key. This will us the `appleboy/ssh-action` action to complete. Those steps combined are highlighted below.
```
- name: Deploy to Server 1
        uses: easingthemes/ssh-deploy@main
        env:
          SSH_PRIVATE_KEY: ${{ secrets.EC2_SSH_KEY }}
          REMOTE_HOST: ${{ secrets.HOST_DNS }}
          REMOTE_USER: ${{ secrets.USERNAME }}
          TARGET: ${{ secrets.TARGET_DIR }}

      - name: Executing remote ssh commands using ssh key
        uses: appleboy/ssh-action@master
        with:
          host: ${{ secrets.HOST_DNS }}
          username: ${{ secrets.USERNAME }}
          key: ${{ secrets.EC2_SSH_KEY }}
          script: |
            sudo apt-get -y update
            sudo apt-get install -y apache2
            sudo systemctl start apache2
            sudo systemctl enable apache2
            cd home
            sudo mv * /var/www/html
```

The YAML file in here can be seen here:
```
name: Push-to-EC2

# Trigger deployment only on push to main branch
on:
  push:
    branches:
      - main

jobs:
  deploy:
    name: Deploy to EC2 on main branch push
    runs-on: ubuntu-latest

    steps:
      - name: Checkout the files
        uses: actions/checkout@v4

      - name: Deploy to Server 1
        uses: easingthemes/ssh-deploy@main
        env:
          SSH_PRIVATE_KEY: ${{ secrets.EC2_SSH_KEY }}
          REMOTE_HOST: ${{ secrets.HOST_DNS }}
          REMOTE_USER: ${{ secrets.USERNAME }}
          TARGET: ${{ secrets.TARGET_DIR }}

      - name: Executing remote ssh commands using ssh key
        uses: appleboy/ssh-action@master
        with:
          host: ${{ secrets.HOST_DNS }}
          username: ${{ secrets.USERNAME }}
          key: ${{ secrets.EC2_SSH_KEY }}
          script: |
            sudo apt-get -y update
            sudo apt-get install -y apache2
            sudo systemctl start apache2
            sudo systemctl enable apache2
            cd home
            sudo mv * /var/www/html
```

After commiting the changes to the file, it will automatically begin the job. This can be viewed under the Actions section in the repository.

While the action is deploying, it will be yellow. Once finished, it will either be red (failed) or green (success). If the action is successful, the resume can be viewed using the Public IP address or the Public DNS name and pasting into a browser. 

The purpose of this set up using the GitHub actions is to create a pipeline that allows for the deployment of changes in the file to be automated. To test this, you can edit the html file residing in your GitHub repository and commit the changes. Once the commit has been pushed, you can check the Actions tab and notice that it has already begun the redploy to include the new changes. After finsihing, the changes should be present with a refresh.

With that said, the project is complete and the resume is able to be shared!
