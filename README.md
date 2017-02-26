## Задание: 
## На host-машине развернуть tarantool/cloud на кластере из 3х виртуальных машинах с Ubuntu.  

########################################
## Part 1 - окружение и Инсталляция  ###
########################################
> Перечислить IP-адреса виртуальных машин и установленную ОС. 
> Предоставить bash-скрипт, который нужно выполнить на host-машине, 
> чтобы запустить облако.
     
    echo 'смотрим окружение'   
    lsb_release -a

>  'мой-хост№1: Ubuntu 14.04.5 LTS (GNU/Linux 3.13.0-110-generic x86_64)'

    echo 'создаем переменную host1 для скрипта app.lua'
    hostname --all-ip-addresses
    export $host1=PUT_ADDRESS_HERE

> 'ip моего хоста: IP address for eth0: 90.156.141.158' $host1=90.156.141.158

    echo 'Ставим docker из репозитария'
    sudo apt-get -y install docker-engine

    echo 'Ставим docker composer'
    cd /opt
    sudo curl -L "https://github.com/docker/compose/releases/download/1.11.1/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose
    sudo chmod +x /usr/local/bin/docker-compose

    echo 'Клонируем репозиторий для теста '
    cd /opt
    sudo git clone https://github.com/tarantool/cloud.git

> Собираем docker cloud (композером) и запускаем как демон

    sudo cd cloud && docker-compose -f docker-compose.yml up -d



##################################
##Part2 - check ##
##################################
> Как убедиться, что облако работает? Например, зайти по такому-то адресу, посмотреть в лог и пр.

    echo 'Посмотрим запущенные контейнеры, занятые порты, и ошибки docker'
    docker ps 
    netstat -apn| grep LISTENING
    cat /var/log/upstart/docker.log | grep error
     
    echo 'Запустилась ли web UI?'
    curl -v http://localhost:5061
    
    echo 'Смотрим в браузере: http://localhost:5061 '
    lynx http://localhost:5061'


############################
## Part 3 - $host1=90.156.141.158  ##
############################

> Предоставить bash-скрипт, который через CLI (taas)
> создает одну логическую ноду в кластере, 
> коннектится к ней, 
> создает в ней space и 
> делает три любых записи


**Запускаем *taas* и создaем кластер *GE* с пользователем *replicator** через app.lua*

    export TARANTOOL_CLOUD_HOST=localhost:5061
    ./taas run --name myinstance tarantool
    ./taas ps

>af8f8ff4038841e292c1e111af351c55  2                          tarantool  500      0        Down      172.55.128.7  3301     unix     2017-02-26
>af8f8ff4038841e292c1e111af351c55  1                          tarantool  500      0        Down      172.55.128.6  3301     unix     2017-02-26

Коннектимся в контейнер и либо выполняем команды вручную, либо выходим и пишем скрипт.
    
    docker ps
    docker exec -t -i af8f8ff4038841e292c1e111af351c55 /bin/sh
    exit

Чтобы не выполнять команды вручную, создаем скрипт app.lua. 


    tee cfg/app.lua <<- EOF   
    box.cfg{listen = 3301,
    logger = docker.log}
    box.schema.user.create('replicator', {password = 'password'})
    box.schema.user.grant('replicator','execute','role','replication')
    box.space._cluster:select({0}, {iterator = 'GE'})
    EOF

Выполняем app.lua в tarantool-контейнере

    ./taas -v deploy af8f8ff4038841e292c1e111af351c55 cfg


#######################
#Part 3 - $host2=172.31.27.169##
#######################

>  подключаем хост2 *ip=172.31.27.169* в кластер и настраиваем репликацию 
>  Повторяем шаги Part2 вносим изменения, добавляя $host1=90.156.141.158

    box.cfg{
      listen = 3301,
      logger = docker.log,
      replication_source = 'replicator:password@90.156.141.158:3301'
    }
    box.space._cluster:select({0}, {iterator = 'GE'})

В логах при выполении select должен появиться id контейнера (sha256) первого хоста

#######################
#Part 4 -Replication ##
#######################


> Результаты host1:cmd3 и host2:cmd3 должны совпадать и должны быть такими-то



**проверим репликацию введем некоторые данные в tarantool:**

host1:

    docker ps
    docker exec -t -i af8f8ff4038841e292c1e111af351c55 /bin/sh
    tarantool

    box.schema.space.create('tarantools')
    box.space.tarantools:create_index('primary', {type = 'tree', parts = {1, 'UNSIGNED'}})
    box.space.tarantools.select(box.space.tarantools,{})
    box.space.tarantools:auto_increment{'Lycosa', 'Lycosa anclata'}

host2: 

    docker ps
    docker exec -t -i af8f8ff4038841e292c1e111af351c55 /bin/sh
    tarantool    
    box.space.tarantools:auto_increment{'Lycosa', 'Lycosa accurata'}
    
host1:
    
    box.space.tarantools:select{}

В логах по команде select должны высветится два элемента 

##################
#Part 5 -хост 3 ##
##################
> Добавить третью виртуальную машину в кластер. Назовем её host3
>  Повторяем шаги Part 2-4, изменения коснуться параметра replication_source

    box.cfg{ listen = 3301,
    	logger = docker.log,
    	replication_source = {replicator:password@90.156.141.158:3301}, {replicator:password@172.31.27.169:3301} }
    box.space._cluster:select({0}, {iterator = 'GE'})

host3:
    
    box.space.tarantools:select{}

В логах должны высветится три элемента 

##################
#Part 6 - логи  ##
##################

    find -f docker.log


########################
#Part 7 создаем ноду ##
########################
>  'создаем ноду'

    ./taas -H localhost:5061 run --name myinstance


----------
##  Итоговые скрипты:
Скрипты в папке cfg 

host1.lua  - скрипт выполняемый на первом хостере

host2.lua  - скрипт выполняемый на втором хостере

host3.lua  - скрипт выполняемый на третьем хостере
