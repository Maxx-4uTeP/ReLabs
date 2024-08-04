# CPH 06: Распределение по ЦОД
##### [Оглавление](../README.md)

### Задание:
Создайте RGW и выдайте bucket через s3.

##### Для этого Вам предстоит выполнить следующие действия:

1. Поднимите RGW realm=ceph.local, rgw-zone=ceph.local, все остальное defaults, машины для размещения шлюзов — ceph1 и ceph2.
*создайте реалм;
*создайте одну зону;
*создайте одну группу;
*коммит изменений и разворот шлюзов;
2. Создайте пользователя test, скопируйте блок с его УЗ на клиентскую машину в файл /root/answer. Формат:
```
     "user": "test",
     "access_key": "84UL8EYOSWXXAAOIQ23J",
     "secret_key": "mCglxWpeMFCIojEkSngO0HjheHVduQTH5sHCQfoO" 
```
3. На клиенте установите необходимый софт (awscli boto boto3 через pip install).
4. Под рутом сконфигурируйте клиент, используя полученные от кластера ключи с профилем ceph (--profile=ceph).
5. Под рутом проверьте настройки и доступ к гейтвею. Если все работает, создайте бакет test.
6. Создайте файл testfile, запишите в него строку "S3 is Working". Скопируйте файл в бакет test.

#### Первоначальная проверка:
* Проверка пула rgw: FAILED
* Проверка создания realm: FAILED
* Проверка создания zonegroup: FAILED
* Проверка создания zone: FAILED
* Проверка создания пользователя: FAILED
* Проверка наличия aws credentials на машине client: FAILED
* Проверка наличия aws config на машине client: FAILED
* Проверка наличия aws bucket на машине client: FAILED

```bash
cat /etc/hosts
```
        10.129.0.19 ceph1.local ceph1
        10.129.0.15 ceph2.local ceph2
        10.129.0.35 ceph3.local ceph3
        10.129.0.5 client.local client


## Практика:
#### RADOS gateway
#### На Ceph есть realm — своего рода доменная сущность для объектного хранилища с зонами и группами. Для теста мы создадим один Realm, одну зону (дефолтную) и одну группу. Если бы мы создавали multisite rgw, то у нас бы было много зон и групп.

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
            "access_key": "8SKFJGA0KVH1QRXE9CZU",
            "secret_key": "Qe0yOsdtGA0j90JVwt34psfbqjaw0rfEeb8ZfWzA"
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
AWS Access Key ID [None]: 8SKFJGA0KVH1QRXE9CZU (то, что скопировали ранее)
AWS Secret Access Key [None]: Qe0yOsdtGA0j90JVwt34psfbqjaw0rfEeb8ZfWzA (то, что скопировали ранее)_
Default region name [None]: _(не нужно заполнять)_
Default output format [None]: json
```
#### Смотрим, все ли у нас хорошо получилось:
```bash
cat ~/.aws/credentials
```
#### Создаем bucket (минимальная ячейка хранения для хранилища, совместимого с S3):
```bash
aws --profile=ceph --endpoint=http://ceph1 s3api create-bucket --bucket test
```
#### Проверяем, создан ли бакет:
```bash
aws --profile=ceph --endpoint=http://ceph1 s3 ls
```
#### Теперь давайте создадим любой файл и скопируем его в хранилище:
```bash
echo "S3 is Working">testfile
aws --profile=ceph --endpoint=http://ceph1 s3 cp testfile s3://test/
```
#### Проверяем, скопировался ли файл:
```bash
aws --profile=ceph --endpoint=http://ceph1 s3 ls test
```

