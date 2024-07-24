# CPH 04: CephFS и объектное хранилище
##### [Оглавление](../README.md)

# Задание:
#### Создайте пул для CEPHFS, выдайте FS и смонитируйте ее на клиент.

##### Для выполнения задания необходимо выполнить следующие шаги:

* Создайте пул CEPHFS c именем volume1 и задеплойте MDS. (ceph fs volume create volume1 и ceph orch apply mds volume1 3)
* Проверьте количество PG \ PGP в пуле и измените их на 32 (ceph osd pool get ... pgp_num pg_num).
* Сгенерируйте ключ для volume1. (ceph fs authorize volume1 client.user / rw)
* На клиенте 1 смонтируйте FS в директорию /test, предварительно создав директорию. (mount -t ceph ceph1,ceph2,ceph3:/ /test -o name=user,secret=<ключ>)
* На клиенте 2 смонтируйте FS в директорию /test, предварительно создав директорию.
* Создайте на клиенте 1 в директории test файл test с содержимым "This is a node 1".
* На клиенте 2 допишите в конец файла /test/test "This is a node 2".

#### Файловое хранилище будет отличаться от блочного RBD, который делали в прошлом задании.
Задаем volume, по сути это root cephfs с именем volume1.
Далее разворачивается 3 mds для отказоустойчивости.
На сервере делаем:
```bash
ceph osd pool ls
ceph osd pool create volume1 32 32
ceph osd pool ls
ceph osd pool application enable volume1 Lab4
ceph osd pool set noautoscale
ceph fs volume create volume1
ceph orch apply mds volume1 3
```

#### CephFS поднял два пула — один пул используется для метаданных, а второй для данных:
```bash
ceph osd lspools
2 cephfs.volume1.meta
3 cephfs.volume1.data
```

В Ceph есть такая функция — автоскейлер. То есть при создании он сам выставит нам минимальное количество PG и при необходимости увеличит их число.
#### Смотрим статус автоскейлера:
```bash
ceph osd pool autoscale-status
```

>[root@ceph1 ~]# ceph osd pool autoscale-status
>POOL                   SIZE  TARGET SIZE  RATE  RAW CAPACITY   RATIO  TARGET RATIO  EFFECTIVE RATIO  BIAS  PG_NUM  NEW PG_NUM  AUTOSCALE  BULK   
>.mgr                 452.0k                3.0        61428M  0.0000                                  1.0       1              off        False  
>volume1                  0                 3.0        61428M  0.0000                                  1.0      32              off        False  
>cephfs.volume1.meta  32768                 3.0        61428M  0.0000                                  4.0      32              off        False  
>cephfs.volume1.data      0                 3.0        61428M  0.0000                                  1.0      32              off        True   




В нашем примере он включен, и мы руками не задавали значение pg. Если же нужно изменить количество pg — так можно увеличить их на кластере. 
```bash
ceph osd pool set cephfs.volume1.data pg_num 32
ceph osd pool set cephfs.volume1.data pgp_num 32
```
При этом параметры будут стремиться поменяться в некоторых случаях. Pапретить такое поведение (для лабораторки) можно командой:
```bash 
ceph osd pool set noautoscale
```




#### Теперь стоит показать, что можно создавать второй рут CephFS. Нужно создать новый пул, создать новую fs с именем cephfs — и создастся новый рут. Сколько бы рутов ни было, mds будут использоваться те же самые, и они все работают в отказоустойчивом режиме.
```bash
ceph osd pool create cephfs_data 32 32
ceph osd pool create cephfs_metadata 32 32
ceph fs new cephfs cephfs_metadata cephfs_data
ceph orch apply mds cephfs 3
```

#### Чаще всего хватает одного root.
#### Делаем пользователя, который ограничен в правах для CephFS:
```bash
ceph fs authorize volume1 client.user / rw | sudo tee ./ceph.client.user.keyring
cat ./ceph.client.user.keyring
```

#### Механизм создания keyring у CephFS и RBD похож, но команды все же отличаются.
#### Переходим на клиент и монтируем (не забываем скопировать secret с сервера):

Client1:
```bash
sudo -i
mkdir /test
mount -t ceph ceph1,ceph3:/ /test -o name=user,secret=AQDHoaBmi9yMBRAAuonsvbr80o8HRzkkwuuRag==
touch /test/test
echo "This is a node 1">/test/test


Client2:
sudo -i
mkdir /test
mount -t ceph ceph1,ceph3:/ /test -o name=user,secret=AQDHoaBmi9yMBRAAuonsvbr80o8HRzkkwuuRag==
echo "This is a node 2">>/test/test


проверить: 
cat /test/test