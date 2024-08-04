# CPH 10: Интеграция с Kubernetes и мониторинг Ceph
##### [Оглавление](../README.md)

### Задание:
Создайте пул на кластере ceph. Ключ доступа интегрируйте k8s c ceph - выдайте диск тестовому приложению.

Создайте пул rbd с именем kube и ключ для пользователя kube. на kubernetes.
Установите helm репо ceph-csi (helm repo add ceph-csi https://ceph.github.io/csi-charts).
На машине k3s поправьте файл в папке rbd/rbd.yml блок. csiConfig:
clusterID: "6c1cc102-e97c-11ec-930c-080027c10f66" monitors:
"v2:<локальный ip-адрес 1 машины кластера>:3300/0,v1:<локальный ip-адрес 1 машины кластера>:6789/0"
"v2:<локальный ip-адрес 2 машины кластера>:3300/0,v1:<локальный ip-адрес 2 машины кластера>:6789/0"
"v2:<локальный ip-адрес 3 машины кластера>:3300/0,v1:<локальный ip-адрес 3 машины кластера>:6789/0"
Информация о вашем кластере содержится в файле /etc/ceph/ceph.conf на первой ноде кластера.
На машине k3s с помощью файла rbd.yaml и helm chart ceph-csi/ceph-csi-rbd установите компоненты ceph в k3s с созданием неймспейса. helm upgrade -i ceph-csi-rbd ceph-csi/ceph-csi-rbd -f rbd/rbd.yml -n ceph-csi-rbd --create-namespace

На машине k3s в файл в папке rbd/secret.yaml запишите полученный от ceph ключ для пользователя kube и примените его, создавая секрет. kubectl apply -f rbd/secret.yaml

На машине k3s в файл в папке rbd/storageclass.yaml поменяйте clusterID: 6c1cc102-e97c-11ec-930c-080027c10f66 на свой и примените его для создания storage class. kubectl apply -f rbd/storageclass.yaml

На машине k3s cоздайте namespace apps. kubectl create namespace apps

На машине k3s c помощью файла rbd/pvc.yaml создайте диск в неймспейсе apps. kubectl apply -f rbd/pvc.yaml -n apps

#### Первоначальная проверка:
Проверка наличия PersistentVolume: FAILED
Проверка наличия PersistentVolumeClaim: FAILED
Проверка наличия StorageClass: FAILED
Проверка наличия диска kube на кластере ceph: FAILED


## Практика
Kubernetes
В работе с контейнерами в Kubernetes необходимо использовать самые свежие образы. Особенно если вы работаете с образами, которые лежат в открытом доступе в Интернете.

Разворачиваем на одной машине Kubernetes, в нашем случае k3s.

У Вас он будет уже установлен.

Ставим kubectl и helm:
```bash
curl -LO https://storage.googleapis.com/kubernetes-release/release/v1.24.0/bin/linux/amd64/kubectl

curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3

chmod 700 get_helm.sh

./get_helm.sh
```
Делаем make k3s_install

```bash
# содержимое мейкфайла
HOST_IP=10.129.0.9 # НЕ ЗАБЫТЬ ВЗЯТЬ СВОЙ IP из ip a
HOST=k3s
KUBECTL=kubectl --kubeconfig ~/.kube/config
k3s_install: # Ниже должны быть символы <TAB> а не пробелы...
        ssh ${HOST} 'export INSTALL_K3S_EXEC=" --disable servicelb --disable traefik"; \
        curl -sfL https://get.k3s.io | sh -'
        scp ${HOST}:/etc/rancher/k3s/k3s.yaml .
        mkdir -p ~/.kube
        sed -r 's/(\b[0-9]{1,3}\.){3}[0-9]{1,3}\b'/"${HOST_IP}"/ k3s.yaml > ~/.kube/config && rm k3s.yaml
```
Далее
```bash
mv ~/.kube/k3s-vm-config ~/.kube/config
```
Смотрим:
```bash
kubectl get all -A Если все ОК
```
Ставим ингресс:
```bash
helm repo add ceph-csi https://ceph.github.io/csi-charts
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/master/deploy/static/provider/baremetal/deploy.yaml
```
Патчим:
```bash
cat > ingress.yaml &lt;<EOF 
spec:
  template:
    spec:
      hostNetwork: true
EOF
kubectl patch deployment ingress-nginx-controller -n ingress-nginx --patch "$(cat ingress.yaml)"
```
Проверяем:
```bash
curl localhost
```
На этом установка кластера k3s окончена.

Переходим в Ceph и начинаем с того, что создаем пул Kubernetes:
```bash
ceph osd pool create kube 32 32
```
Далее необходимо сделать пользователя:
```bash
ceph auth get-key client.kube
```
Он нам создал ключ для пользователя Kubernetes, который мы будем использовать в дальнейшем, например: 
```
AQAsRA5jk2MJDBAAi2hBQ80Y0gsELiTv/NCxtw==[
```
! На Kubernetes должны быть установлены драйвера CEPH (ceph-common).

Переходим на Kubernetes.

Ставим провайдер:
```bash
vi rbd1.yml
```
В него надо установить IP мониторов, ID CEPH и несколько значений меняем на true, а именно podSecurityPolicy на nodeplugin и на provisioner.

С помощью helm inspect можно выкачивать свежий values-файл:
```bash
helm inspect values ceph-csi/ceph-csi-rbd > rbd1.yml
```
Дальше мы пишем “helm upgrade” с нашим “rbd1.yml” и устанавливаем Provisioner:
```bash
helm upgrade -i ceph-csi-rbd ceph-csi/ceph-csi-rbd -f rbd.yml -n ceph-csi-rbd --create-namespace
```
Можем проверить командой kubectl get all -A.

Дальше правим “secret.yaml” (ключи должны быть те, что получили на Ceph):
```bash
vi rbd/secret.yaml
```
Устанавливаем секрет:
```bash
kubectl apply -f rbd/secret.yaml
```
Смотрим “storageclass”:
```bash
cat rbd/storageclass.yaml
```
ID должен совпадать с тем, что получен с Сeph.
```bash
Fstype - ext4, так как понимается всеми Linux.
```
Устанавливаем storageclass:
```bash
kubectl apply -f rbd/storageclass.yaml
```
Делаем namespace apps:
```bash
kubectl create ns apps
```
Привязываем PVC к неймспейсу и смотрим, создаалась ли PVC:
```bash
kubectl apply -f rbd/pvc.yaml -n apps
kubectl get pv -A
```
Также можно посмотреть на Ceph, что создался rbd, командой rbd ls -p kube или через веб-интерфейс во вкладке Block > Images.

Для теста мы берем контейнер Caroline (открытый сервис, написанный на Go)
```bash
cat caroline/Deployment.yaml
kubectl apply -f caroline/Deployment.yaml
kubectl get all -A
```
Создадим для нее сервис:
```bash
kubectl apply -f caroline/Service.yaml
```
И создадим ингресс:
```bash
kubectl apply -f caroline/Ingress.yaml
```
К caroline мы можем обратиться по следующему адресу: carolyne.moonstreet.local (который указывает на ip-адрес Ingress).

Смотрим, как называется ее POD, через kubectl get pod -n apps (там находим carolyne и копируем название — в нашем случае carolyne-544f6b5bd-96wkd).

Идем в POD:
```bash
kubectl exec -it --namespace=apps carolyne-544f6b5bd-96wkd -- sh
```
Через команду ls видим диск. По команде df можем посмотреть объем диска.

Каролина сама ничего не пишет, поэтому можем сами что-нибудь записать, например, файл размером в 500 MB:
```bash
cd disk
dd if=/dev/zero of=daygeek2.txt bs=500M count=1
```
Caroline при этом работает.

Monitoring
Ниже представлен Ceph Dashboard. Все, что мы видим — информация из Grafana. В первую очередь смотрим Status, состояние quorum, количество OSD.

![7](https://img.rebrainme.com/workshops/ceph/master/d56a52a6-7ebe-4311-939f-52116edb2aac/98686548-0e32-4493-b9e0-abb0fa530bab/10_07.png)

В разделе Cluster > Monitoring > Alerts лежат все оповещения о событиях. Если хочется добавить что-то свое, придется очень долго ковырять настройки.

Все активные Alert собираются в Cluster > Monitoring > Active Alerts. Это единственный у него приемник по умолчанию, чтобы посмотреть где и что случилось.

Мы можем выйти в Alertmanager, настроить мы можем только режим тишины (когда не сообщать Alerts). Все остальное — через правку конфигов.

В Grafana все те же самые Dashboard, которые отображаются и в Ceph Dashboard.

В самой Grafana можно дополнительно посмотреть Host Details, который показывает загрузку CPU, памяти, сетевую загрузку. То есть, можно просто смотреть, без дополнительных настроек.

CEPH Dashboard, и Grafana, и все остальное полностью настраивается. В Ceph Dashboard можно подцепить свои сертификаты, а можно их выключить. Можно настроить внешнюю Grafana. Но только имейте в виду, что когда будете настраивать внешнюю Grafana, то все Dashboard должны так и называться, чтобы он их смог подтянуть.


