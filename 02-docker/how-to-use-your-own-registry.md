Одна из особенностей платформы Docker — простота использования образов из централизованного хранилища. [Docker Hub](https://hub.docker.com/) — это ведущий репозиторий образов с тысячами официальных образов, готовых к использованию. Вы также можете легко загрузить свои образы в реестр Docker Hub, чтобы все могли воспользоваться преимуществами ваших Docker-приложений.

Но в некоторых сценариях вам может не потребоваться отправлять образы за пределы брандмауэра. В этом случае вы можете настроить локальный реестр, используя проект с открытым исходным кодом [Distribution](https://github.com/distribution/distribution) . В этой статье мы рассмотрим настройку локального экземпляра проекта Distribution, где ваши команды смогут обмениваться образами с помощью уже знакомых им команд Docker: `docker push`и `docker pull`.

### Предпосылки

Для выполнения этого урока вам понадобится следующее:
- Бесплатная учетная запись Docker
    - Вы можете [зарегистрировать бесплатную учетную запись Docker](https://hub.docker.com/) и получить неограниченное количество бесплатных публичных репозиториев.
- Docker работает локально
    - [Инструкции по загрузке и установке Docker](https://docs.docker.com/desktop/)

### Запуск службы распространения

Проект дистрибутива упакован как [официальный образ на Docker Hub](https://hub.docker.com/_/registry) . Чтобы запустить версию локально, выполните следующую команду:

`$ docker run -d -p 5000:5000 --name registry registry:2.7`

Этот `-d`флаг запустит контейнер в отсоединённом режиме. `-p`Он открывает порт 5000 в сети вашего локального компьютера. С помощью этого флага мы также присваиваем контейнеру имя . Подробнее об этих флагах и всех флагах команды можно узнать `--name`в нашей [документации](https://docs.docker.com/engine/reference/run/)`docker run` .

### Выталкивание и извлечение из местного реестра

Теперь, когда наш реестр работает локально, давайте проследим за журналами контейнера, чтобы убедиться, что наш образ загружается и извлекается локально:

`$ docker logs -f registry`

Откройте другой терминал и скачайте [официальный образ Ubuntu](https://hub.docker.com/_/ubuntu) из Docker Hub. Мы используем его в нашем примере ниже:

`$ docker pull ubuntu`

Чтобы загрузить или извлечь данные из локального реестра, необходимо добавить расположение реестра к имени репозитория. Формат следующий: `my.registry.address:port/repositoryname`.

В нашем примере нам нужно заменить `my.registry.address.port`на , `localhost:5000`поскольку наш реестр работает на локальном хосте и прослушивает порт 5000. Вот полное имя репозитория: `localhost:5000/ubuntu`. Для этого выполним `docker tag`команду:

`$ docker tag ubuntu localhost:5000/ubuntu`

Теперь мы можем отправить данные в наш локальный реестр.

`$ docker push localhost:5000/ubuntu`

_ПРИМЕЧАНИЕ:_

_Docker ищет либо «.» (разделитель домена), либо «:» (разделитель порта), чтобы определить, что первая часть имени репозитория — это местоположение, а не имя пользователя. Если у вас просто localhost без .localdomain или :5000 (подойдёт любой из вариантов), Docker посчитает localhost именем пользователя, как в localhost/ubuntu или samalba/hipache. Затем он попытается выполнить отправку в реестр по умолчанию, которым является Docker Hub. Наличие точки или двоеточия в первой части указывает Docker, что это имя содержит имя хоста, и что следует выполнить отправку в указанное вами местоположение._

Вернитесь в терминал, где отслеживаются логи нашего реестра. В журналах вы увидите записи, содержащие запрос на сохранение образа Ubuntu:

```
...
172.17.0.1 - - [26/Feb/2021:18:10:57 +0000] "POST /v2/ubuntu/blobs/uploads/ HTTP/1.1" 202 0 "" "docker/20.10.2 go/go1.13.15 git-commit/8891c58 kernel/4.19.121-linuxkit os/linux arch/amd64 UpstreamClient(Docker-Client/20.10.2 \\(darwin\\))"

172.17.0.1 - - [26/Feb/2021:18:10:57 +0000] "POST /v2/ubuntu/blobs/uploads/ HTTP/1.1" 202 0 "" "docker/20.10.2 go/go1.13.15 git-commit/8891c58 kernel/4.19.121-linuxkit os/linux arch/amd64 UpstreamClient(Docker-Client/20.10.2 \\(darwin\\))"

172.17.0.1 - - [26/Feb/2021:18:10:57 +0000] "POST /v2/ubuntu/blobs/uploads/ HTTP/1.1" 202 0 "" "docker/20.10.2 go/go1.13.15 git-commit/8891c58 kernel/4.19.121-linuxkit os/linux arch/amd64 UpstreamClient(Docker-Client/20.10.2 \\(darwin\\))"
...
```

Теперь давайте удалим наш `localhost:5000/ubuntu`образ, а затем извлечем его из локального репозитория, чтобы убедиться, что все работает правильно.

Сначала распечатаем список изображений, которые есть у нас локально:

```
$ docker images
REPOSITORY  TAG       IMAGE ID      CREATED           SIZE
registry    2.7       5c4008a25e05  40 hours ago      26.2MB
ubuntu      latest    f63181f19b2f  5 weeks ago       72.9MB
localhost:5000/ubuntu latest f63181f19b2f 5 weeks ago 72.9MB
```

Теперь удалим `localhost:5000/ubuntu:latest`образ с нашей локальной машины:

```
$ docker rmi localhost:5000/ubuntu
Untagged: localhost:5000/ubuntu:latest
Untagged: localhost:5000/ubuntu@sha256:3093096ee188f8...8c091c8cb4579c39cc4e
```

Давайте еще раз проверим, удалено ли изображение:

```
$ docker images
REPOSITORY   TAG       IMAGE ID       CREATED        SIZE
registry     2.7       5c4008a25e05   40 hours ago   26.2MB
ubuntu       latest    f63181f19b2f   5 weeks ago    72.9MB
```

Наконец извлеките образ из нашего локального реестра и убедитесь, что теперь он загружен в наш локальный экземпляр Docker.

```
$ docker pull localhost:5000/ubuntu
Using default tag: latest
latest: Pulling from ubuntu
Digest: sha256:sha256:3093096ee188f8...8c091c8cb4579c39cc4e
Status: Downloaded newer image for localhost:5000/ubuntu:latest
localhost:5000/ubuntu:latest

$ docker images
REPOSITORY              TAG       IMAGE ID       CREATED        SIZE
registry                2.7       5c4008a25e05   40 hours ago   26.2MB
ubuntu                  latest    f63181f19b2f   5 weeks ago    72.9MB
localhost:5000/ubuntu   latest    f63181f19b2f   5 weeks ago    72.9MB
```

### Краткое содержание

В этой статье мы рассмотрели запуск локального реестра образов. Мы также извлекли образ для Docker Hub, пометили его тегом в нашем локальном реестре, а затем отправили его в наш локальный реестр, запустив Distribution.

Если вы хотите узнать больше о проекте Distribution, перейдите на [домашнюю страницу проекта с открытым исходным кодом на GitHub](https://github.com/distribution/distribution) и обязательно ознакомьтесь с [документацией](https://github.com/distribution/distribution/tree/main/docs) .

https://www.docker.com/blog/how-to-use-your-own-registry-2/