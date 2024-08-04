# CPH 12: Финальное задание Ceph
##### [Оглавление](../README.md)
### Описание:
Надо починить ceph-кластер, который поломался на предприятии, и тем самым восстановить работу рабочей базы. Ваш напарник не нашел ничего лучше, чем "застопорить кластер" для сохранения данных. К счастью вы недавно подготовили новый OSD-узел, но не вводили его в работу. Необходимо восстановить работоспособность кластера, а затем подготовить S3-бакет для работы с файлами.
##### Дано:
* 3 машины c поднятым кластером Ceph. На ceph3 поврежден диск.
* Заранее подготовленный OSD-узел ceph4.
* Машина клиента client.
### Задание:
1. В кластере Ceph на мастер-узле проверить флаг паузы и снять его.
2. Восстановить работу кластера:
    * Подготовить машину ceph4 к вводу в кластер;
    * Добавить на ceph4 ssh-ключ;
    * Добавить в кластер хост ceph4;
    * Добавить диск ceph4 в кластер;
    * Найти сбойный диск и удалить его из crushmap и с osd и соответствующий демон.
3. Проверить, что кластер заработал.
4. Поднять RGW realm=ceph.local rgw-zone=ceph.local все остальное defaults (radosgw-admin). Машины для размещения шлюзов: ceph1 и ceph2.
    * Создать realm;
    * Создать zonegroup;
    * Создать zone;
    * Произвести коммит изменений и соркестрировать контейнеры на ceph1 и ceph2.
    * Создать пользователя test (radosgw-admin).
5. На клиенте client установить необходимый софт awscli, boto, boto3 (pip install).
    * Сконфигурировать клиент client, используя полученные от кластера ключи - профиль ceph (aws configure);
    * Проверить настройки и доступ к гейтвею: если все работает, создать s3-бакет backup;
    * Создать файл test в s3-бакете backup.

#### Первоначальная проверка:
* Пауза с кластера убрана: FAILED
* Работы дисков: FAILED
* Проверка пула rgw: FAILED
* Проверка установки boto на клиенте: FAILED
* Проверка установки awscli на клиенте: FAILED
* Проверка наличия файла credentials на клиенте: FAILED
* Проверка наличия файла config с указанным output-типом json на клиенте: FAILED
* Проверка наличия файла test в bucket backup на клиенте: FAILED


  > 10.129.0.28 ceph1.local ceph1
  10.129.0.25 ceph2.local ceph2
  10.129.0.27 ceph3.local ceph3
  10.129.0.9 client.local client
  10.129.0.31 ceph4.local ceph4


#### Поднимаем стенд, смотрим статус:
Видим что одна OSD в состоянии down
```bash
[root@ceph1 ~]# ceph -s
  cluster:
    id:     52969ac2-5253-11ef-8416-d00db853addd
    health: HEALTH_WARN
            pauserd,pausewr flag(s) set
            1 osds down
            1 host (1 osds) down
            Reduced data availability: 8 pgs inactive
            Degraded data redundancy: 26/78 objects degraded (33.333%), 17 pgs degraded

  services:
    mon: 3 daemons, quorum ceph1,ceph3,ceph2 (age 112s)
    mgr: ceph1.zfcllt(active, since 3m), standbys: ceph3.hgqdre, ceph2.qtfffz
    osd: 3 osds: 2 up (since 58s), 3 in (since 2m)
         flags pauserd,pausewr

  data:
    pools:   2 pools, 33 pgs
    objects: 26 objects, 15 MiB
    usage:   904 MiB used, 59 GiB / 60 GiB avail
    pgs:     96.970% pgs not active
             26/78 objects degraded (33.333%)
             16 undersized+peered
             16 undersized+degraded+peered
             1  active+undersized+degraded
```

#### Выясняем какая OSD умерла:

```bash
[root@ceph1 ~]# ceph osd tree
ID  CLASS  WEIGHT   TYPE NAME       STATUS  REWEIGHT  PRI-AFF
-1         0.05846  root default
-3         0.01949      host ceph1
 0    hdd  0.01949          osd.0       up   1.00000  1.00000
-5         0.01949      host ceph2
 1    hdd  0.01949          osd.1       up   1.00000  1.00000
-7         0.01949      host ceph3
 2    hdd  0.01949          osd.2     down   1.00000  1.00000
```

#### Выясняем IP на 4 хосте и прописываем его в хосты.

```bash
ip a # на 4 сервере
echo "10.129.0.31 ceph4.local ceph4" >> /etc/hosts
```

#### Копирувем SSH ключ на 4 сервер
```bash
ssh-copy-id -f -i /etc/ceph/ceph.pub root@ceph4
```

#### На четвертой ноде поставили Ceph-common, обменялись с ней ключиками. Добавляем хост:
```bash
ceph orch host add ceph4 10.129.0.31
```
#### Добавляем с Ceph 4 его диск:
```bash
ceph orch daemon add osd ceph4:/dev/vdb
```
#### Далее удаляем OSD2 диск, который уже “поломан”:
```bash
ceph osd rm osd.2
```
#### Тот же диск находится в crush map, поэтому надо сначала удалить этот диск из карты crush:
```bash
ceph osd crush remove osd.2
ceph orch daemon rm osd.2 --force
```
#### Снимаем кластер с паузы:
```bash
ceph osd unset pause
```
#### Проверяем статус. Кластер починился:
```bash
[root@ceph1 ~]# ceph -s
  cluster:
    id:     52969ac2-5253-11ef-8416-d00db853addd
    health: HEALTH_OK

  services:
    mon: 3 daemons, quorum ceph1,ceph3,ceph2 (age 25m)
    mgr: ceph1.zfcllt(active, since 27m), standbys: ceph3.hgqdre, ceph2.qtfffz
    osd: 3 osds: 3 up (since 8m), 3 in (since 9m)

  data:
    pools:   2 pools, 33 pgs
    objects: 26 objects, 15 MiB
    usage:   905 MiB used, 59 GiB / 60 GiB avail
    pgs:     33 active+clean
```

#### Cоздаем realm, зону и группу:
```bash
radosgw-admin realm create --rgw-realm=ceph.local --default
radosgw-admin zonegroup create --rgw-zonegroup=default --master --default
radosgw-admin zone create --rgw-zonegroup=default --rgw-zone=ceph.local --master --default
```
#### Делаем commit (radosgw-admin — это не команда управления хранилищем, это команда создания объектного gateway):
```bash
radosgw-admin period update --rgw-realm=ceph.local --commit
```
#### Оркестрируем контейнеры — размещаем на 1 и 2 ноде:
```bash
ceph orch apply rgw ceph --realm=ceph.local --zone=ceph.local --placement="2 ceph1 ceph2"
```
#### Делаем пользователя тест:
```bash
radosgw-admin user create --display-name="test" --uid=test
```
#### Из всего вывода нас интересуют 3 строки в ключе “keys”:
```
    "keys": [
        {
            "user": "test",
            "access_key": "BJ29MHNGY8V5QI49FZV1",
            "secret_key": "hZOieIuABF9sqy3gO0GbGNUFOaP4CM4JLL63NCyD"
        }
```
#### Их копируем в отдельный файл, чтобы не забыть.
#### Командой podman ps смотрим, что rgw ceph развернулся.
#### Далее переходим на клиента. Здесь через pip ставим awscli (работать будем по протоколу s3):
```bash
ssh client
apt-get update
apt-get install python3-pip -y
pip3 install awscli boto boto3
```
#### Конфигурируем awscli:
```bash
aws configure --profile=ceph
```
```
AWS Access Key ID [None]: BJ29MHNGY8V5QI49FZV1 (то, что скопировали ранее)
AWS Secret Access Key [None]: hZOieIuABF9sqy3gO0GbGNUFOaP4CM4JLL63NCyD (то, что скопировали ранее)_
Default region name [None]: _(не нужно заполнять)_
Default output format [None]: json
```
#### Смотрим, все ли у нас хорошо получилось:
```bash
cat ~/.aws/credentials
```
#### Создаем bucket (минимальная ячейка хранения для хранилища, совместимого с S3):
```bash
aws --profile=ceph --endpoint=http://ceph1 s3api create-bucket --bucket backup
```
#### Проверяем, создан ли бакет:
```bash
aws --profile=ceph --endpoint=http://ceph1 s3 ls
```
#### Теперь давайте создадим любой файл и скопируем его в хранилище:
```bash
echo "S3 is Working">test
aws --profile=ceph --endpoint=http://ceph1 s3 cp test s3://backup/
```
#### Проверяем, скопировался ли файл:
```bash
aws --profile=ceph --endpoint=http://ceph1 s3 ls backup
```


#### Итоговая проверка:
* Пауза с кластера убрана: OK
* Работы дисков: OK
* Проверка пула rgw: OK
* Проверка установки boto на клиенте: OK
* Проверка установки awscli на клиенте: OK
* Проверка наличия файла credentials на клиенте: OK
* Проверка наличия файла config с указанным output-типом json на клиенте: OK
* Проверка наличия файла test в bucket backup на клиенте: OK


