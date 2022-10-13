# Домашнее задание к занятию "6.5. Elasticsearch"

## Задача 1

В этом задании вы потренируетесь в:
- установке elasticsearch
- первоначальном конфигурировании elastcisearch
- запуске elasticsearch в docker

Используя докер образ [centos:7](https://hub.docker.com/_/centos) как базовый и 
[документацию по установке и запуску Elastcisearch](https://www.elastic.co/guide/en/elasticsearch/reference/current/targz.html):

- составьте Dockerfile-манифест для elasticsearch
- соберите docker-образ и сделайте `push` в ваш docker.io репозиторий
- запустите контейнер из получившегося образа и выполните запрос пути `/` c хост-машины

Требования к `elasticsearch.yml`:
- данные `path` должны сохраняться в `/var/lib`
- имя ноды должно быть `netology_test`

В ответе приведите:
- текст Dockerfile манифеста
- ссылку на образ в репозитории dockerhub
- ответ `elasticsearch` на запрос пути `/` в json виде

Подсказки:
- возможно вам понадобится установка пакета perl-Digest-SHA для корректной работы пакета shasum
- при сетевых проблемах внимательно изучите кластерные и сетевые настройки в elasticsearch.yml
- при некоторых проблемах вам поможет docker директива ulimit
- elasticsearch в логах обычно описывает проблему и пути ее решения

Далее мы будем работать с данным экземпляром elasticsearch.

## Решение
Готовим `Dockerfile` вида
```
FROM centos:7
RUN cd /opt && \
    groupadd elasticsearch && \
    useradd -c "elasticsearch" -g elasticsearch elasticsearch &&\
    yum update -y && yum -y install wget perl-Digest-SHA && \
    wget https://artifacts.elastic.co/downloads/elasticsearch/elasticsearch-8.4.3-linux-x86_64.tar.gz && \
    wget https://artifacts.elastic.co/downloads/elasticsearch/elasticsearch-8.4.3-linux-x86_64.tar.gz.sha512 && \
    shasum -a 512 -c elasticsearch-8.4.3-linux-x86_64.tar.gz.sha512 && \
    tar -xzf elasticsearch-8.4.3-linux-x86_64.tar.gz && \
	rm elasticsearch-8.4.3-linux-x86_64.tar.gz elasticsearch-8.4.3-linux-x86_64.tar.gz.sha512 && \ 
	mkdir /var/lib/data && chmod -R 777 /var/lib/data && \
	chown -R elasticsearch:elasticsearch /opt/elasticsearch-8.4.3 && \
	yum clean all
USER elasticsearch
WORKDIR /opt/elasticsearch-8.4.3/
COPY elasticsearch.yml  config/
EXPOSE 9200 9300
ENTRYPOINT ["bin/elasticsearch"]
```

Используем `elasticsearch.yml`
```YAML
node:
  name: netology_test
path:
  data: /var/lib/data
xpack.ml.enabled: false
```

```bash
⋊> ~/DZ6.5 curl --insecure -u elastic https://localhost:9200
{
  "name" : "netology_test",
  "cluster_name" : "elasticsearch",
  "cluster_uuid" : "utpzFmK8SxiQ39LlKQ1u1A",
  "version" : {
    "number" : "8.4.3",
    "build_flavor" : "default",
    "build_type" : "tar",
    "build_hash" : "42f05b9372a9a4a470db3b52817899b99a76ee73",
    "build_date" : "2022-10-04T07:17:24.662462378Z",
    "build_snapshot" : false,
    "lucene_version" : "9.3.0",
    "minimum_wire_compatibility_version" : "7.17.0",
    "minimum_index_compatibility_version" : "7.0.0"
  },
  "tagline" : "You Know, for Search"
}

```

Ссылка на образ

[Docker Hub](https://hub.docker.com/r/vkuzevanov/vkelastic/)


## Задача 2

В этом задании вы научитесь:
- создавать и удалять индексы
- изучать состояние кластера
- обосновывать причину деградации доступности данных

Ознакомтесь с [документацией](https://www.elastic.co/guide/en/elasticsearch/reference/current/indices-create-index.html) 
и добавьте в `elasticsearch` 3 индекса, в соответствии со таблицей:

| Имя | Количество реплик | Количество шард |
|-----|-------------------|-----------------|
| ind-1| 0 | 1 |
| ind-2 | 1 | 2 |
| ind-3 | 2 | 4 |

Получите список индексов и их статусов, используя API и **приведите в ответе** на задание.

Получите состояние кластера `elasticsearch`, используя API.

Как вы думаете, почему часть индексов и кластер находится в состоянии yellow?

Удалите все индексы.

**Важно**

При проектировании кластера elasticsearch нужно корректно рассчитывать количество реплик и шард,
иначе возможна потеря данных индексов, вплоть до полной, при деградации системы.

## Решение

```bash
⋊> ~/DZ6.5 curl -X PUT --insecure -u elastic "https://localhost:9200/ind-1?pretty" -H 'Content-Type: application/json' -d'
           {
             "settings": {
               "index": {
                 "number_of_shards": 1,  
                 "number_of_replicas": 0 
               }
             }
           }
           '
{
  "acknowledged" : true,
  "shards_acknowledged" : true,
  "index" : "ind-1"
}
⋊> ~/DZ6.5 curl -X PUT --insecure -u elastic "https://localhost:9200/ind-2?pretty" -H 'Content-Type: application/json' -d'
           {
             "settings": {
               "index": {
                 "number_of_shards": 2,  
                 "number_of_replicas": 1 
               }
             }
           }
           '
{
  "acknowledged" : true,
  "shards_acknowledged" : true,
  "index" : "ind-2"
}
⋊> ~/DZ6.5 curl -X PUT --insecure -u elastic "https://localhost:9200/ind-3?pretty" -H 'Content-Type: application/json' -d'
           {
             "settings": {
               "index": {
                 "number_of_shards": 4,  
                 "number_of_replicas": 2 
               }
             }
           }
           '
{
  "acknowledged" : true,
  "shards_acknowledged" : true,
  "index" : "ind-3"
}
```

Cписок индексов и их статусов

```bash
⋊> ~/DZ6.5 curl -X GET --insecure -u elastic "https://localhost:9200/_cat/indices?v=true"
health status index uuid                   pri rep docs.count docs.deleted store.size pri.store.size
green  open   ind-1 lVfWGhi5T0eBWkJhDRLaMg   1   0          0            0       225b           225b
yellow open   ind-3 sb5MyVQsRAi44xEvX8bJCA   4   2          0            0       900b           900b
yellow open   ind-2 HxJ0aMYnSluVbu-Zxz7Cvg   2   1          0            0       450b           450b
```
Cостояние кластера elasticsearch
```bash
⋊> ~/DZ6.5 curl -X GET --insecure -u elastic "https://localhost:9200/_cluster/health?pretty"
{
  "cluster_name" : "elasticsearch",
  "status" : "yellow",
  "timed_out" : false,
  "number_of_nodes" : 1,
  "number_of_data_nodes" : 1,
  "active_primary_shards" : 9,
  "active_shards" : 9,
  "relocating_shards" : 0,
  "initializing_shards" : 0,
  "unassigned_shards" : 10,
  "delayed_unassigned_shards" : 0,
  "number_of_pending_tasks" : 0,
  "number_of_in_flight_fetch" : 0,
  "task_max_waiting_in_queue_millis" : 0,
  "active_shards_percent_as_number" : 47.368421052631575
}
```
Индексы и кластер находятся в yellow, так как при создании индексов мы указали количество реплик > 1. Поскольку у нас всего 1 нода, реплицировать индексы некуда :(

Удаление индексов

```bash
⋊> ~/DZ6.5 curl -X DELETE --insecure -u elastic "https://localhost:9200/ind-1?pretty"

{
  "acknowledged" : true
}
⋊> ~/DZ6.5 curl -X DELETE --insecure -u elastic "https://localhost:9200/ind-2?pretty"

{
  "acknowledged" : true
}
⋊> ~/DZ6.5 curl -X DELETE --insecure -u elastic "https://localhost:9200/ind-3?pretty"

{
  "acknowledged" : true
}
```

## Задача 3

В данном задании вы научитесь:
- создавать бэкапы данных
- восстанавливать индексы из бэкапов

Создайте директорию `{путь до корневой директории с elasticsearch в образе}/snapshots`.

Используя API [зарегистрируйте](https://www.elastic.co/guide/en/elasticsearch/reference/current/snapshots-register-repository.html#snapshots-register-repository) 
данную директорию как `snapshot repository` c именем `netology_backup`.

**Приведите в ответе** запрос API и результат вызова API для создания репозитория.

Создайте индекс `test` с 0 реплик и 1 шардом и **приведите в ответе** список индексов.

[Создайте `snapshot`](https://www.elastic.co/guide/en/elasticsearch/reference/current/snapshots-take-snapshot.html) 
состояния кластера `elasticsearch`.

**Приведите в ответе** список файлов в директории со `snapshot`ами.

Удалите индекс `test` и создайте индекс `test-2`. **Приведите в ответе** список индексов.

[Восстановите](https://www.elastic.co/guide/en/elasticsearch/reference/current/snapshots-restore-snapshot.html) состояние
кластера `elasticsearch` из `snapshot`, созданного ранее. 

**Приведите в ответе** запрос к API восстановления и итоговый список индексов.

Подсказки:
- возможно вам понадобится доработать `elasticsearch.yml` в части директивы `path.repo` и перезапустить `elasticsearch`

## Решение


- Создали директорию /opt/elasticsearch-8.4.3/snaps
- В конфигурационный файл elasticsearch.yml внесли параметр path.repo: ["/opt/elasticsearch-8.4.3/snaps"]
- Рестартовали контейнер


Запрос на регистрацию snapshot репозитория

```bash
⋊> ~/DZ6.5 curl -X PUT --insecure -u elastic"https://localhost:9200/_snapshot/netology_backup?pretty" -H 'Content-Type: application/json' -d'
           {
             "type": "fs",
             "settings": {
               "location": "/opt/elasticsearch-8.4.3/snaps"
             }
           }
           '
{
  "acknowledged" : true
}
```

Создали индекс `test`
```bash
⋊> ~/DZ6.5 curl -X PUT --insecure -u elastic"https://localhost:9200/test?pretty" -H 'Content-Type: application/json' -d'
           {
             "settings": {
               "index": {
                 "number_of_shards": 1,  
                 "number_of_replicas": 0 
               }
             }
           }
           '
⋊> ~/DZ6.5 curl -X GET --insecure -u elastic"https://localhost:9200/_cat/indices?v=true"
health status index uuid                   pri rep docs.count docs.deleted store.size pri.store.size
green  open   test  h_iRtHMER-GPx4ULM33OPA   1   0          0            0       225b           225b
```

Сделали snapshot кластера
```bash
⋊> ~/DZ6.5 curl -X PUT --insecure -u elastic"https://localhost:9200/_snapshot/netology_backup/my_snapshot1?pretty"
{
  "accepted" : true
}


[elasticsearch@09c46adf8509 elasticsearch-8.4.3]$ ls -l /opt/elasticsearch-8.4.3/snaps/
total 36
-rw-r--r-- 1 elasticsearch elasticsearch  1097 Oct 13 17:32 index-0
-rw-r--r-- 1 elasticsearch elasticsearch     8 Oct 13 17:32 index.latest
drwxr-xr-x 5 elasticsearch elasticsearch  4096 Oct 13 17:32 indices
-rw-r--r-- 1 elasticsearch elasticsearch 16647 Oct 13 17:32 meta-tqBi5wgNRZqlMnjOLoAnCw.dat
-rw-r--r-- 1 elasticsearch elasticsearch   387 Oct 13 17:32 snap-tqBi5wgNRZqlMnjOLoAnCw.dat

```

Удалили индекс test и создали индекс test-2
```bash
⋊> ~/DZ6.5 curl -X GET --insecure -u elastic"https://localhost:9200/_cat/indices?v=true"
health status index  uuid                   pri rep docs.count docs.deleted store.size pri.store.size
green  open   test-2 d25OsBslSnSlmROWpU2gwA   1   0          0            0       225b           225b
```

Cписок доступных snapshotов

```bash 
⋊> ~/DZ6.5 curl -X GET --insecure -u elastic"https://localhost:9200/_snapshot/netology_backup/*?verbose=false&pretty"
{
  "snapshots" : [
    {
      "snapshot" : "my_snapshot1",
      "uuid" : "tqBi5wgNRZqlMnjOLoAnCw",
      "repository" : "netology_backup",
      "indices" : [
        ".geoip_databases",
        ".security-7",
        "test"
      ],
      "data_streams" : [ ],
      "state" : "SUCCESS"
    }
  ],
  "total" : 1,
  "remaining" : 0
}
```

Восстановили состояние кластера из snapshot, созданного ранее

```bash
⋊> ~/DZ6.5 curl -X POST --insecure -u elastic"https://localhost:9200/_snapshot/netology_backup/my_snapshot1/_restore?pretty" -H 'Content-Type: application/json' -d'
           {
             "indices": "*",
             "include_global_state": true
            }
            '

{
  "accepted" : true
}

curl -X POST --insecure -u elastic"https://localhost:9200/_snapshot/netology_backup/my_snapshot1/_restore?pretty" -H 'Content-Type: application/json' -d'
{
  "indices": "*",
  "include_global_state": true
 }
 '
```

Итоговый список индексов
```bash
⋊> ~/DZ6.5 curl -X GET --insecure -u elastic"https://localhost:9200/_cat/indices?v=true"
health status index  uuid                   pri rep docs.count docs.deleted store.size pri.store.size
green  open   test-2 d25OsBslSnSlmROWpU2gwA   1   0          0            0       225b           225b
green    open   test   OgA7YWRgQWK766XUS7_cjQ   1   0  
```
