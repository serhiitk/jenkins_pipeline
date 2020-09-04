
# Jenkins-CI pipeline
Jenkins-CI pipeline builds an image from a *Dockerfile*, run image in the test environment (http://jenkins.example.com:8081), and push the image to the Nexus repository.

## Variables
Variables defines in `./vars/default_vars.yml`

    java_version: 'openjdk-11-jdk'
    ....
    jenkins_host_ip: ...
    gitlab_host_ip: ...
    nexus_host_ip: ...
    ....
    and etc.

## Prerequisites
### Preparation GitLab-server
- create a user for Jenkins-CI projects: `jenkins-user`
- create project for Jenkins-CI pipeline: `jenkins-test`

### Preparation Nexus-server repository
- create repository: **jenkins-test-app1** (parameters: docker hosted, **https port - 18093**, enable Docker API)
- create Role with "admin" and "view" privileges to **jenkins-test-app1** repository (example: Jenkins-CI Role)
- create repository user: **jenkinsci-user**. And grant Role created in the previous step 

## Environment setup
- start `environment_setup.yml` to perform the next tasks:
     - to Jenkins server copy Nexus SSL-certificate for connection between docker on Jenkins and Nexus server. 
     - to Jenkins server resolve local DNS names (gitlab, nexus and jenkins servers) by adding hosts to `/etc/hosts` file.
     - to GitLab server resolve local DNS names (gitlab, nexus and jenkins servers) by adding hosts to `/etc/hosts` file.

           $ ansible-playbook environment_setup.yml

## Deploy Jenkins-CI Server
- start deployment Jenkins-CI (Jenkins default admin user and password - `admin` ) : 

      $ ansible-playbook deploy_jenkins.yml

### Configure GitLab-to-Jenkins authentication  ( https://github.com/jenkinsci/gitlab-plugin#gitlab-to-jenkins-authentication )
#### Configuring per-project authentication:
- **in Jenkins** generate and copy 'Secret Token' ( jenkins job -> Configure -> section "Build when a change is pushed to GitLab" -> click 'Advanced' -> 'Secret Token' field -> click 'Generate' )   
- **in GitLab** create a *Webhook* for your project ( gitlab project -> Settings -> Webhooks -> enter the trigger URL: `http://jenkins.example.com:8080/project/jenkins-test-pipeline` -> Secret Token: `<generated token>` )

### Configure Jenkins-to-GitLab authentication ( https://github.com/jenkinsci/gitlab-plugin#jenkins-to-gitlab-authentication )
- **in GitLab** generate and copy 'Access Token' (jenkins-user -> Settings -> Click on 'Access Tokens' -> Create a token named e.g. *'JenkinsAPIToken'* with 'api' scope; expiration is optional  )
- **in Jenkins** add a 'GitLab API token' credential ( Manage Credentials -> Add Credentials -> Kind: `'GitLab API token'` , API token: `<generated 'JenkinsAPIToken'>` , ID: `JenkinsAPIToken` )
- **in Jenkins** configure GitLab Connections ( Configure System -> section GitLab -> Connection name: `local-Gitlab` , Gitlab host URL: `http://gitlab.example.com` , Credentials: `JenkinsAPIToken`)

### Configure GitLab repository credentials (for cloning)
- **in Jenkins** add credentials to local-Gitlab ( Manage Credentials -> Add Credentials -> Kind: `'Username with password'` , Username: `jenkins-user` , Password: `<jenkins-user password>` , ID: `gitlab_user` )

### Configure Nexus repository credentials (for pushing images)
- **in Jenkins** add credentials to Nexus Repository ( Manage Credentials -> Add Credentials -> Kind: `'Username with password'` , Username: `jenkinsci-user` , Password: `<jenkinsci-user password>` , ID: `nexus_user` )

## Configure pipeline variables in `Jenkinsfile`
       
      def CFG_NEXUS_HOST_NAME = "nexus.example.com"
      def CFG_NEXUS_REPOSITORY_PORT = "18093"
      def CFG_NEXUS_REPOSITORY_NAME = "jenkins-test-app1"
      ... ... ...      
      and etc.

## Push test project from ./jenkins/repository_files/ to the created repository on your local GitLab server
    $ git remote add origin http://gitlab.example.com/jenkins-user/jenkins-test.git
    $ git push -u origin master

## Backing up Jenkins home directory

    $ ansible-playbook backup_jenkins.yml

Archive Jenkins settings and plugins:
- `$JENKINS_HOME/*.xml`
- `$JENKINS_HOME/*.key*`
- `$JENKINS_HOME/jobs/*`
- `$JENKINS_HOME/nodes/*`
- `$JENKINS_HOME/plugins/*.[hj]pi`
- `$JENKINS_HOME/secrets/*`
- `$JENKINS_HOME/userContent/*`
- `$JENKINS_HOME/users/*`
