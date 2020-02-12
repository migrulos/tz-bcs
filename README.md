## Основное задание
### - Платформа minikube развернута локально

На хост-машине нужно не меньше 4 гигабайт оперативной памяти, чтобы нормально отработал post-install hook db-init.job.yaml (для нормального завершения sentry upgrade нужно не меньше 3Гб):
https://github.com/helm/charts/issues/15296
https://github.com/helm/charts/issues/10291
(ключи для minikube start --memory='4000mb' --cpus=2)


### - Деплой приложения Sentry производится с помощью Helm-чартов

В качестве базового использовать будем этот чарт: https://github.com/helm/charts/tree/master/stable/sentry


### - База данных для приложения - любая поддерживаемая

"Sentry does not support any database besides PostgreSQL." (источник: https://docs.sentry.io/server/faq/)
В качестве БД используется Postgress, плюс Redis в качестве брокера сообщений.


### - Пароли от БД должны храниться в секретах(Kind: Secrets)

Пароль шифруется и кладётся в secrets тут: sentry/charts/postgresql/templates/secrets.yaml
Берётся он из переменной postgresql.password, которая определяется в sentry/charts/postgresql/templates/_helpers.tpl и является либо тем, что мы объявим в sentry/values.yaml или в sentry/charts/postgresql/values.yaml, либо результатом функции randAlphaNum

проверяем:
kubectl get secret --namespace default sentry-1-sentry-postgresql

| NAME        | TYPE           | DATA  | AGE |
| ------------- |:-------------:| -----:| -----:|
| sentry-1-sentry-postgresql  |Opaque | 1  |    52m 

посмотреть сам пароль можно так:
kubectl get secret --namespace default sentry-1-sentry-postgresql -o jsonpath="{.data.postgresql-password}" | base64 --decode


### - Доступ к веб-морде приложения получить любым способом(ingress, NodePort, etc.)

Используем NodePort (порт возьмём из стандартного port-range):
values.yaml:
```
service:
  name: sentry
  type: NodePort
  nodePort: 30000
```

доступ:
Get the application URL by running these commands:  
```                                                          
  export NODE_PORT=$(kubectl get --namespace default -o jsonpath="{.spec.ports[0].nodePort}" services sentry-1)  
  export NODE_IP=$(kubectl get nodes --namespace default -o jsonpath="{.items[0].status.addresses[0].address}")  
  echo http://$NODE_IP:$NODE_PORT/auth/login/sentry                                                              
```

### - Выставить ограничение ресурсов приложения в соответствии с документацией.

В Requirements для self-hosted установки (https://docs.sentry.io/server/installation/) указаны следующие требования:
```
At least 2400MB memory
2 CPU Cores
```
по факту памяти лучше поставить больше, чтобы sentry upgrade отработал, в разных issue пишут то 3, то 4Гб (https://github.com/helm/charts/issues/15296, https://github.com/helm/charts/issues/10291)
host-машина должна соответствовать этим требованиям (ключи для minikube start --memory='4000mb' --cpus=2)

Для хука db-init выставлены ограничения по памяти, т.к. для процесса миграции (происходит также при первичной инициализации базы, sentry upgrade) нужно не меньше 3Гб ОЗУ
values.yaml:
```
hooks:                                                                                                                      
  affinity: {}                                                                                                              
  dbInit:                                                                                                                   
    resources:                                                               
      limits:                                                                                                               
        memory: 3200Mi                                                                                                      
      requests:                                                                                                             
        memory: 3000Mi
```
Для основных сервисов sentry (web, worker, cron) рекомендаций по ограничению ресурсов в официальной документации не нашёл. По сути это один и тот же docker-image, только с разными коммандами внутри 
(sentry run cron, sentry run worker).
note: Web, worker и cron - я называю "сервисами" не в контексте k8s, а в контексте sentry, как они и сами их называют, например, здесь: https://docs.sentry.io/server/upgrading/#restarting-services

Посмотрим сколько ресурсов потребляют поды через несколько минут после запуска:
поставим metrics-server:  minikube addons enable metrics-server
```
kubectl top pod -n default
NAME                               CPU(cores)   MEMORY(bytes)            
sentry-1-cron-596dc9c4dc-s96kd     2m           102Mi                    
sentry-1-sentry-postgresql-0       4m           35Mi                     
sentry-1-sentry-redis-master-0     9m           5Mi                      
sentry-1-sentry-redis-slave-0      5m           4Mi                      
sentry-1-sentry-redis-slave-1      7m           4Mi                      
sentry-1-web-cd47cfb67-lz946       1m           343Mi                    
sentry-1-web-cd47cfb67-mbrtk       1m           346Mi                    
sentry-1-worker-695956f8dc-2zlmq   6m           151Mi                    
sentry-1-worker-695956f8dc-gqdjg   4m           145Mi                    
```
Исходя из полученных цифр можно поставить, например, такие ограничения ресурсов (requests - минимально доступные ресурсы при выборе ноды, на которой запускать под, CPU считается в milliCPU):
```
web:
  resources:
    limits:
      cpu: 600m
      memory: 3Gi
    requests:
      cpu: 100m
      memory: 400Mi

cron:
  resources:
    limits:
      cpu: 250m
      memory: 250Mi
    requests:
      cpu: 100m
      memory: 150Mi

worker:
  resources:
    limits:
      cpu: 300m
      memory: 3Gi
    requests:
      cpu: 100m
      memory: 150Mi
```

##  По желанию:
### - Научить контейнеры, которые подключаются к базе, ожидать ее полноценного запуска(порт БД доступен) и только потом запускаться. (Делается через InitContainer)

Подробного описания архитектуры sentry мне найти не удалось (и мейнтенеры рассказывать отказываются: https://github.com/getsentry/sentry/issues/3419).
Поэтому какие именно из контейнеров подключаются к базе - вопрос открытый. В логах подов запросы к БД видны только у worker-ов. Но логи подов - это не debug, и что там под капотом можно увидеть только 
если изучать код. Кроме того, адрес базы, логин и пароль от неё передаются в env переменных всем трём сервисам: web, cron, worker. Поэтому можно добавить InitContainer во все три.

Проверять доступность базы будем с помощью команды pg_isready, которую будем запускать из контейнера на базе официального образа postgres:12
Добавляем в deployment:
```
initContainers:
- name: check-pg
  image: postgres:12
  command: ['sh', '-c', 'until pg_isready -h {{ template "sentry.postgresql.host" . }} -p {{ template "sentry.postgresql.port" . }} -U {{ default "sentry" .Values.postgresql.postgresqlUserame}}; do echo waiting for database; sleep 2; done;']
```

Проверяем, что наш InitContainer работает:
```
kubectl logs sentry-1-web-5dbf968b98-fgjpn -c check-pg -f          

sentry-1-sentry-postgresql:5432 - no response
waiting for database
....
sentry-1-sentry-postgresql:5432 - accepting connections
```


### - Т.к. сервис является командным и востребованным, его даунтайм недопустим, то учесть "систему" бекапов бд. (Это может быть обычный скрипт в другом контейнере, главное – stateful)


В идеале тут стоит написать скрипт, который бы полученный дамп кроме сохранения в самом кубере перекидывал его (предварительно заархивировав) ещё куда-нибудь на удалённый сервер, например, по scp. Упаковать этот скрипт в отдельный докер-образ и запускать уже его, передавая ему через env-переменные db-host, port username и password.
Но т.к. поднятие удалённого сервера выходит за рамки данного ТЗ, то сделаю сохранение дампа через persistentVolumeClaim.

Используем CronJob, который запускаетяся раз в сутки (тут опционально - в зависимости от необходимой частоты бэкапа и имеющегося места на дисках легко настраивается нужный shedule), поднимает image postgres, в котором выполняет pg_dumpall

описание CronJob тут: **sentry/template/pg-dump.yaml**

проверяем, что дамп пишется:
watch -n 1 kubectl get pods - видим как cronjob запускает поды и отключает их после выполнения
kubectl get persistentvolumes - ищем нужный pv
kubectl describe pv pv-name - смотрим куда он пишет (source.Path: /tmp/hostpath-provisioner/pvc-e4173330-8299-4447-bf3e-b79695e79ab6)
ls -l /tmp/hostpath-provisioner/pvc-e4173330-8299-4447-bf3e-b79695e79ab6 - файлы на месте


### - Zero-downtime метод обновления Sentry(RollingUpdate Strategy)

Сервис Cron рекомендуется запускать в единственном экземпляре, поэтому для него RollingUpdate делать не будем:
It’s recommended to only run one of them at the time or you will see unnecessary extra tasks being pushed onto the queues (Источник: https://docs.sentry.io/server/queue/#starting-the-cron-process)

Для сервисов web и worker поставим по 2 реплики и включим strategy.type:RollingUpdate с самой простой стратегией (сначала поднимается 1 новый pod - maxSurge: 1, только после этого гасится один из старых - maxUnavailable: 0)
```
values.yaml:
web:
  replicacount: 2
worker:
  replicacount: 2

web-deployment.yaml:
spec:
  replicas: {{ .Values.web.replicacount }}
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 0
      maxSurge: 1

workers-deployment.yaml:
spec:
  replicas: {{ .Values.worker.replicacount }}
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 0
      maxSurge: 1
```


### - Интегрировать авторизацию через GitHub\GitLab

Инструкции:
https://docs.sentry.io/server/integrations/github/
https://developer.github.com/apps/building-github-apps/creating-a-github-app/

Полученные при создании приложения на гитхабе данные (id,secret,key) нужно положить в конфиг: sentry/templates/configmap.yaml

### - Если есть возможность, выставить веб-морду наружу (IP или домен) Результатом будут являться готовые для деплоя Helm-чарты и рабочая веб-морда приложения, в которое можно войти(!важно). Можно писать видео или скриншоты экрана. Результат залить на любую систему контроля версий либо отправить обратным письмом.

Явки-пароли в письме.
