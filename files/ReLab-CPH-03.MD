# CPH 03: Возможности Ceph и блочный доступ
##### [Оглавление](../README.md)

#### На сервере делаем пул RBD с именем RBD. Он связывается с application RBD. Этот pool будет отдавать блочку:
```bash
ceph osd pool create rbd 32 32
ceph osd pool application enable rbd rbd
rbd pool init -p rbd
```

#### Мы можем посмотреть пулы командой ceph osd lspools.
#### Создаем диск (lun) на 10 гигабайт и смотрим по нему информацию:
```bash
rbd create disk01 --size 10G --pool rbd
rbd --image disk01 -p rbd info
```

#### Генерируем минимальный конфиг для клиента, он обычно для всех одинаковый. При создании клиентов рекомендуется такой конфиг зашивать на уровне cloud-init \ шаблона VM — он никогда не меняется.
```bash
ceph config generate-minimal-conf > ceph.conf
scp /etc/ceph/ceph.conf client:/etc/ceph/ceph.conf
```

#### Генерируем ключ. Он будет ограниченным, чтобы клиент ходил только на пул RBD.
```bash
ceph auth get-or-create client.rbd mon 'allow r' osd 'allow class-read object_prefix rbd_children, allow rwx pool=rbd'
ceph auth get-or-create client.rbd | ssh root@client sudo tee /etc/ceph/keyring
```

#### Переходим на клиент. Используя ключ доступа, лезем на кластер. Теперь можно управлять кластером с клиента (но не всем: например, если дать команду сделать новые пулы, он этого не сможет).
```bash
ceph -s --name client.rbd

#disk01 на практике видно (командой rbd ls --name client.rbd), и его можно примапить на машину.
sudo rbd map --image disk01 --name client.rbd
rbd showmapped  --name client.rbd
```

#### Замапим его:
```bash
sudo mkdir /test
sudo mount /dev/rbd0 /test/
```

#### LVM:
#### Подготовьте диск через LVM (vg=test lv=test size=1G).
#### LVM не увидит rbd0. Надо будет добавить строчку, чтобы LVM начал понимать диски вида rbd (на машинах уже будет добавлена). 
#### По умолчанию он скажет: «отброшена по фильтрации».
#### Можно также создать диск непосредственно с клиента
```bash
rbd create disk02 --size 1024 --name client.rbd
sudo mkfs.ext4 /dev/rbd0 -L cephbd
sudo rbd map --image disk02 --name client.rbd
lsblk

sudo vgcreate test /dev/rbd1
sudo vgdisplay 
sudo lvcreate -L 1020M -n test test # размер меньше 1Г потому как не помещается  Volume group "test" has insufficient free space (255 extents): 256 required.
sudo lvdisplay # создали LVM 
sudo umount /test/ # Размонтируем предыдущий тест
sudo mkfs.xfs  /dev/test/test -L cephbd # Форматируем LVM /dev/test/test в XFS как требует задание
sudo mount /dev/test/test /test # Монтируем LVM /dev/test/test в директорию /test

sudo touch /test/test # Создаем файлик
sudo chown user /test/test # даем на него права юзеру
echo "This is an RBD - it's working" >>/test/test # Пишем в файлик строку как требует задание
cat /test/test # Проверяем вывод строки из файла

```


