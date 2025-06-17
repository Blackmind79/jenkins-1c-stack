# About
Jenkins project

# Attention
1. JVM options specifically for the Jenkins controller should be set through JENKINS_JAVA_OPTS, as other tools might also respond to the JAVA_OPTS environment variable.
2. At the first run you can get initial admin password in folder `/var/jenkins_home/secrets/initialAdminPassword`
3. For 1C also add plugins: SonarQube Scanner, Allure

# Links
Link      | URL
----------|----
Dockerhub | [Jenkins in Dockerhub](https://hub.docker.com/r/jenkins/jenkins)
Github    | [Jenkins in Github](https://github.com/jenkinsci/docker/blob/master/README.md)

# Add agent

## First step
First of all, create SSH keys (private and public)
```sh
ssh-keygen -t ed25519 -f <filename in current dir> -N <password for opening key> -C "<comment>"
```
Example:
```sh
ssh-keygen -t ed25519 -f agent_key -N "" -C "ssh agent key without password"
```
## Second step
Define the enviroment variable in **.env** file `JENKINS_AGENT_SSH_PUBKEY` with pubkey you create. Run contaners with `docker compose up -d`.
Example of **.env** file in file `dotenv`

## Third step
Enter the master container (`docker exec -it jenkins bash`):
If you dont't create folder **$JENKINS_HOME/.ssh** earlier, prepare it:
- create folder **$JENKINS_HOME/.ssh** (`mkdir $JENKINS_HOME/.ssh && chmod -R 700 $JENKINS_HOME/.ssh`)
- create file **known_hosts** (`touch $JENKINS_HOME/.ssh/known_hosts && chmod -R 600 $JENKINS_HOME/.ssh/known_hosts`)
If it is a stand-alone server for agents, scan keys to **known_hosts**:
- scan keys from agent container (`ssh-keyscan -H ${JENKINS_SSH_AGENT_HOSTNAME_OR_IP} >> ~/.ssh/known_hosts`)
In case of Docker container (not to scan every time) set "Non verifying Verification Strategy" in the menu on the left-hand side of the agent configuration panel in Jenkins Master.

OR

Do it in one-line:
```sh
docker compose up -d jenkins \
  && docker exec -it jenkins bash -c "mkdir $JENKINS_HOME/.ssh && chmod -R 700 $JENKINS_HOME/.ssh && touch $JENKINS_HOME/.ssh/known_hosts && chmod -R 600 $JENKINS_HOME/.ssh/known_hosts" \
  && docker compose up -d jenkins-ssh-agent
```

## Fourth step
Add agent thru Jenkins interface with SSH creds (username `jenkins` and private SSH key configured earlier)

# Update plugins
Get list of currently installed plugins
```sh
JENKINS_HOST=username:password@myhost.com:port
curl -sSL "http://$JENKINS_HOST/pluginManager/api/xml?depth=1&xpath=/*/*/shortName|/*/*/version&wrapper=plugins" | perl -pe 's/.*?<shortName>([\w-]+).*?<version>([^<]+)()(<\/\w+>)+/\1 \2\n/g'|sed 's/ /:/'
```
| Note: you can see examples of installed plugins in file `current_plugins.xml`. So you can create plugins.txt file with list of plugins you need:

>plugins.txt
>```plaintext
>  ssh-credentials
>  mina-sshd-api-common
>  mina-sshd-api-core
>  ssh-slaves
>```

To update plugins
```sh
JENKINS_IMAGE=jenkins/jenkins:lts-jdk17
docker run -it ${JENKINS_IMAGE} bash -c "stty -onlcr && jenkins-plugin-cli -f /usr/share/jenkins/ref/plugins.txt --available-updates --output txt" >  plugins2.txt
mv plugins2.txt plugins.txt
```

# SonarQube
Context             | URL
--------------------|----
Dockerhub           | [SonarQube at Dockerhub](https://hub.docker.com/_/sonarqube)
Github              | [SonarQube at Github](https://github.com/SonarSource/docker-sonarqube)
Github compose.yaml | [SonarQube at Github](https://github.com/SonarSource/docker-sonarqube/blob/master/example-compose-files/sq-with-postgres/docker-compose.yml)

# Default Admins credentials
Must be changed ASAP after installation!
```
Login: admin
Password: admin
```

# Steps after migration
After restore DB to new sql server edition or new sonarqube version (if needed), run manually setup
http://<yourserver>:<port>/setup
https://<your_server>/setup