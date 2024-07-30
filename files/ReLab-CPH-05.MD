# CPH 05: Структура хранения и администрирование хранилища
##### [Оглавление](../README.md)

### Задание:
Создайте пул для RBD и lun в 5GB и выдайте их через iscsi, предварительно установив iscsi-шлюз.

##### Для выполнения задания необходимо выполнить следующие шаги:

* Создайте пул rbd c именем DISK1 и в нем lun 5gb.
* Разверните iscsi gateway (УЗ admin01 passw01 размещение ceph1 ceph2 добавить в доверенные ip и прописать их адреса).
* На первом узле вычислите контейнер iscsi gateway и войдите в него через cephadm enter.
* Используя gwcli, создайте таргет iqn.2022-01.local.ceph.iscsi-gw:iscsi-igw, после чего зацепите шлюзы подключения за ранее созданный диск.
* Не выходя из gwcli, создайте клиента с уз (username=iscsiadmin password=iscsipassword) и дайте ему диск.
* На клиенте поставьте ПО icsci. Настройте клиент на подключение.
* Настройте мультипас драйвер. Правим /etc/multipath.conf
```json
    devices {
            device {
                        vendor                 "LIO-ORG"
                        product                "TCMU device"
                        hardware_handler       "1 alua"
                        path_grouping_policy   "failover"
                        path_selector          "queue-length 0"
                        failback               60
                        path_checker           tur
                        prio                   alua
                        prio_args              exclusive_pref_bit
                        fast_io_fail_tmo       25
                        no_path_retry          queue
                }
        }
```
* Запустите мультипас и демона icsci.
* Отсканируйте и примапьте Lun через iscsi.
* Создайте директорию /test и смонтируйте в нее диск.
    Можете проверить доступность записи на диск копированием на него любых файлов. 

#### Первоначальная проверка:
* Проверка пула iscsi: FAILED
* Проверка размера диска iscsi: FAILED
* Проверка cephadm gateway: FAILED
* Проверка работы контейнера: FAILED
* Проверка создания gateway: FAILED
* Проверка присоединения диска DISK1 в контейнере: FAILED
* Проверка создания пользователя client в контейнере: FAILED
* Проверка присоединения диска DISK1 пользователю client в контейнере: FAILED
* Проверка подключения и вывода утилиты multipath: FAILED
* Проверка примонтирования директории /dev/mapper/mpatha в директорию /test: FAILED


## Практика:
#### Сегодня создаем iscsi gateway.

#### Создаем пул с 32 pg, application c типом RBD и диск с именем “DISK1”:
```bash
ceph osd pool create iscsi 32 32
ceph osd pool application enable iscsi rbd
rbd --pool iscsi create DISK1 --size=5GB
```
#### Разворачиваем iscsi gateway: в trusted_ip_list указываем IP кластерных хостов — посмотреть можно командой cat /etc/hosts, в расположении — ноды 2 и 3:
        10.129.0.12 ceph1.local ceph1
        10.129.0.20 ceph2.local ceph2
        10.129.0.9 ceph3.local ceph3
        10.129.0.21 client.local client
```bash
ceph orch apply iscsi iscsi admin01 passw01 --placement ceph1,ceph2 --trusted_ip_list 10.129.0.12,10.129.0.20

cephadm ls --no-detail
```
#### Вывод:
```json
    {
        "style": "cephadm:v1",
        "name": "iscsi.iscsi.ceph1.nwlvca",
        "fsid": "5308f1fa-4e55-11ef-884d-d00d177569af",
        "systemd_unit": "ceph-5308f1fa-4e55-11ef-884d-d00d177569af@iscsi.iscsi.ceph1.nwlvca"
    }
```
#### Тут надо отметить, что ранее iscsi gw был внешним плагином, поэтому команды по управлению им не ставятся вместе с ceph-common, но они есть в его контейнере.
#### Возможно в дальнейшем это причешут.
name контейнера на примере — “iscsi.iscsi.ceph1.nwlvca”
fsid контейнера на примере — “5308f1fa-4e55-11ef-884d-d00d177569af”.

#### Через ‘Cephadm shell’ можно войти в него. Отсюда уже можно управлять iscsi:
```bash
cephadm enter -n iscsi.iscsi.ceph1.nwlvca
```
#### Запустим оболочку управления iscsi:
```bash
gwcli
```
#### Начнем с имени цели:
```bash
> cd iscsi-targets
> create iqn.2022-01.local.ceph.iscsi-gw:iscsi-igw
```
#### Это все именование в стиле iscsi. Так мы создаем “iqn”. (iSCSI qualified name) — это уникальный идентификатор, назначаемый для каждого iSCSI Target и Initiator.
#### Теперь создадим ему два шлюза:
```bash
> cd iqn.2022-01.local.ceph.iscsi-gw:iscsi-igw/gateways
> create ceph1.local 10.129.0.12 skipchecks=true
> create ceph2.local 10.129.0.20 skipchecks=true
```
#### Теперь мы идем в диски и приаттачиваем ранее созданный диск:
```bash
> cd /disks
> attach iscsi/DISK1
```
#### Отсюда же можно сделать “create disk”, но есть риск, что покажет warning «диск не управляется Ceph» — баг вылезает от версии к версии.
#### Создаем клиента (инициатор):
```bash
> cd /iscsi-targets/iqn.2022-01.local.ceph.iscsi-gw:iscsi-igw/hosts
> create iqn.2022-01.ceph.iscsi:iscsi-client
```
#### Мы зададим ему login и password (для аутентификации):
```bash
> auth username=iscsiadmin password=iscsipassword
```
#### Объявляем диск для клиента:
```bash
> disk add iscsi/DISK1
```
#### Все. Мы создали target \ initiator. Можно давать команду exit.
#### Дальше мы ставим на клиенте софт:
```bash
ssh client
apt install open-iscsi multipath-tools. 
```
#### Идем в /etc/iscsi/iscsid.conf, ставим логин и пароль, который установили, а в /etc/iscsi/initiatorname.iscsi напишем имя инициатора iqn.2022-01.ceph.iscsi:iscsi-client.
#### Правим мультипас-драйвер:
```bash
nano /etc/multipath.conf
```
```json
devices {
    device {
            vendor                 "LIO-ORG"
            product                "TCMU device"
            hardware_handler       "1 alua"
            path_grouping_policy   "failover"
            path_selector          "queue-length 0"
            failback               60
            path_checker           tur
            prio                   alua
            prio_args              exclusive_pref_bit
            fast_io_fail_tmo       25
            no_path_retry          queue
    }
}
```
#### Запускаем дискавери:
```bash
iscsiadm -m discovery -t st -p 10.129.0.12
```
#### Логинимся:
```bash
iscsiadm -m node -T iscsiadm -m node -T iqn.2022-01.local.ceph.iscsi-gw:iscsi-igw -l
```
#### Проверяем:
```bash
multipath -l

lsblk
```
#### Далее монтируем полученный диск в директорию /test.
```bash
mkdir /test
mkfs.xfs /dev/mapper/mpatha
mount /dev/mapper/mpatha /test
```

#### Итоговая проверка:
Проверка пула iscsi: OK
Проверка размера диска iscsi: OK
Проверка cephadm gateway: OK
Проверка работы контейнера: OK
Проверка создания gateway: OK
Проверка присоединения диска DISK1 в контейнере: OK
Проверка создания пользователя client в контейнере: OK
Проверка присоединения диска DISK1 пользователю client в контейнере: OK
Проверка подключения и вывода утилиты multipath: OK
Проверка примонтирования директории /dev/mapper/mpatha в директорию /test: OK