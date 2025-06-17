# О проекте
Данный проект содержит в себе основные программы для CI/CD и проверки кода.

Стэк программ включает в себя: Jenkins, SonarQube, Allure

# ВНИМАНИЕ
Из-за особенности настройки Allure, доступ к UI идет как: http://host.docker.internal:5252, если установлен локально.
Либо укажите внешнее имя хостовой машины, где развернут allure_api.

# Полезные ссылки и документация
Описание              | Ссылка
----------------------|-------
Jenkins (Dockerhub)   | [Jenkins in Dockerhub](https://hub.docker.com/r/jenkins/jenkins)
Jenkins (Github)      | [Jenkins in Github](https://github.com/jenkinsci/docker/blob/master/README.md)
SonarQube (Dockerhub) | [SonarQube at Dockerhub](https://hub.docker.com/_/sonarqube)
SonarQube (Github)    | [SonarQube at Github](https://github.com/SonarSource/docker-sonarqube)
SonarQube (Github compose.yaml) | [SonarQube at Github](https://github.com/SonarSource/docker-sonarqube/blob/master/example-compose-files/sq-with-postgres/docker-compose.yml)

# Основные шаги
0. Предполагается, что docker уже установлен в системе. Если нет, следуйте документации [Get Docker](https://docs.docker.com/get-started/get-docker)
1. Создайте SSH-ключи для подключения агента к Jenkins.
```sh
# ssh-keygen -t ed25519 -f <filename in current dir> -N <password for opening key> -C "<comment>"
# Пример создания ключа без пароля:
ssh-keygen -t ed25519 -f agent_key -N "" -C "ssh agent key without password"
```
2. Запустите проект
- Соберите образ агента `docker compose build`, так как для агента требуется доустановить дополнительные пакеты.
- Укажите в `.env` файле сгенерированный публичный ключ JENKINS_AGENT_SSH_PUBKEY
- Запустите стэк `docker compose up -d`
3. Настройте вход в Jenkins. Первоначальный пароль содержится в контейнере в `/var/jenkins_home/secrets/initialAdminPassword` (можно увидеть в папке проекта ) или в логах контейнера `docker logs jenkins`
4. Добавьте агент в Jenkins, указав такие параметры как:
```
Удалённая корневая директория: /home/jenkins/agent
Способ запуска: Launch agent via SSH
Host Key Verification Strategy: Non verifying Verification Strategy (иначе при перезапуске постоянно будет отваливаться known_host)
Credentials: SSH Username with private key (укажите ваш сгенерированный приватный ключ)
```

>Если у вас для агента используется хостовый сервер, то для безопасности можно сделать привязку через known_host.
> Для этого выполните:
> ```sh
> docker compose up -d jenkins \
>   && docker exec -it jenkins bash -c "mkdir $JENKINS_HOME/.ssh && chmod -R 700 $JENKINS_HOME/.ssh && touch $JENKINS_HOME/.ssh/known_hosts && chmod -R 600 $JENKINS_HOME/.ssh/known_hosts" \
>   && docker compose up -d jenkins-ssh-agent \
>   && docker exec -it jenkins bash -c "ssh-keyscan -H ${JENKINS_SSH_AGENT_HOSTNAME_OR_IP} >> $JENKINS_HOME/.ssh/known_hosts"
> ```
> Где JENKINS_SSH_AGENT_HOSTNAME_OR_IP - адрес вашего хоста для агента

5. При первом запуске поменяйте административный пароль `SonarQube`
```
Login: admin
Password: admin
```
6. Добавьте плагины в Jenkins такие как: SonarQube Scanner, Allure
7. Добавьте плагины в SonarQube такие как: Russian Pack, 1C (BSL) Community Plugin, YAML Analyzer, ShellCheck Analyzer

# Обновление версии SonarQube
Для обновления версии SonarQube, после изменения тэга образа, требуется запустить обновление.
Зайдите в браузере на ваш сервер с суффиксом `/setup`:
```
http://<yourserver>:<port>/setup
https://<your_server>/setup
```
Далее следуйте сообщениям мастера обновления.


# Дополнительно
Параметры JVM, предназначенные специально для контроллера Jenkins, должны быть установлены через JENKINS_JAVA_OPTS,
поскольку другие инструменты также могут реагировать на переменную окружения JAVA_OPTS.

