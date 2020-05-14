# Работа с Managed Kubernetes и Container Registry

### Подготовка окружения
* Зайдите в консоль облака https://console.cloud.yandex.ru и создайте себе каталог (folder) - создавать сеть по умолчанию там не требуется.
* Установить Yandex CLI в рабочую папку Jenkins
* В терминале рабочей станции инициируйте `yc init`.
* Выберите созданный вами каталог.


### Создание сети для задания

Создадим сеть и подсети в трех зонах доступности для работы группы виртуальных машин

```
yc vpc network create --name yc-auto-network

zones=(a b c)

for i in ${!zones[@]}; do
  echo "Creating subnet yc-auto-subnet-$i"
  yc vpc subnet create --name yc-auto-subnet-$i \
  --zone ru-central1-${zones[$i]} \
  --range 192.168.$i.0/24 \
  --network-name yc-auto-network
done
```



### Создание кластера Kubernetes и Container Registry

#### Создадим сервисный аккаунт для кластера
```
FOLDER_ID=$(yc config get folder-id)

yc iam service-account create --name k8s-sa-${FOLDER_ID}
SA_ID=$(yc iam service-account get --name k8s-sa-${FOLDER_ID} --format json | jq .id -r)
yc resource-manager folder add-access-binding --id $FOLDER_ID --role admin --subject serviceAccount:$SA_ID
```

#### Создадим мастер
```

yc managed-kubernetes cluster create \
 --name k8s-demo --network-name yc-auto-network \
 --zone ru-central1-a  --subnet-name yc-auto-subnet-0 \
 --public-ip \
 --service-account-id ${SA_ID} --node-service-account-id ${SA_ID} --async

```
Создание мастера занимает около 7 минут - в это время мы создадим Container Registry и загрузим в него Docker образ

Создадим Container Registry

```
yc container registry create --name yc-auto-cr
```

Аутентифицируемся в Container Registry

```
yc container registry configure-docker
```

Создадим Dockerfile

```
cat > demo.dockerfile <<EOF
FROM demo-app
COPY demo-app /usr/src/myapp
WORKDIR /usr/src/myapp
CMD ["java", "-jar" "demo-0.0.1.jar"]
EOF
```
Соберем образ и загрузим его в Registry
```
REGISTRY_ID=$(yc container registry get --name yc-auto-cr  --format json | jq .id -r)
docker build . -f hello.dockerfile \
-t cr.yandex/$REGISTRY_ID/ubuntu:hello

docker push cr.yandex/${REGISTRY_ID}/ubuntu:hello
```
Проверим, что в Container Registry появился созданный образ

```
yc container image list
```


#### Создание группы узлов

Перейдите в веб интерфейс вашего каталога в раздел "Managed Service For Kubernetes". Дождитесь создания кластера - он должен перейти в статус `Ready` и состояние `Healthy`.
Теперь создадим группу узлов

```
yc managed-kubernetes node-group create \
 --name k8s-demo-ng \
 --cluster-name k8s-demo \
 --platform-id standard-v2 \
 --public-ip \
 --cores 2 \
 --memory 4 \
 --disk-type network-ssd \
 --location subnet-name=yc-auto-subnet-0,zone=ru-central1-a \
 --async
 ```

#### Подключение к кластеру

Создание группы узлов занимает около 3 минут - давайте пока подключимся к кластеру при помощи kubectl.

Настроим аутентификацию в кластере
```
yc managed-kubernetes cluster get-credentials --external --name k8s-demo
```

Дождемся создания группы узлов с помощью kubectl
```
watch kubectl get nodes
```
Когда команда начнет выводить 2 узла в статусе `Ready`, значит кластер готов для работы. 
Нажмите  `Ctrl`+`C` для выхода из режима watch.


### Создаем configfile для развертывания с помощью jenkins: deploymen.yml и service.yml

* deploymen.yml:
```
apiVersion: apps/v1 # for versions before 1.9.0 use apps/v1beta2
kind: Deployment
metadata:
  name: demo-app
spec:
  selector:
    matchLabels:
      app: demo-app
  replicas: 1 # tells deployment to run 2 pods matching the template
  template:
    metadata:
      labels:
        app: demo-app
    spec:
      containers:
      - name: demo-app
        image: cr.yandex/crpan8upoull02qji09s/openjdk8:demo
        ports:
        - containerPort: 80
```

* service.yml:
```
apiVersion: v1
kind: Service
metadata:
  name: demo-service
spec:
  type: LoadBalancer
  selector:
    app: demo-app # < --- Вот тут мы говорим сервису искать контейнеры с такими метаданнами(тэгами)
  ports:
    - protocol: TCP
      port: 80
      targetPort: 8080
```

### Создаем Jenkinsfile для Multibranch pipeline

* Jenkinsfile:
```
pipeline {

  environment {
    PROJECT = "demo"
  }
  agent any
  
  
  stages {
    stage('Deploy demo') {
      // Developer Branches
      steps {

          // Create namespace if it doesn't exist
          sh("export KUBECONFIG=/var/lib/jenkins/test.kubeconfig")
          sh("/var/lib/jenkins/yandex-cloud/bin/yc container registry configure-docker")
          sh("/var/lib/jenkins/yandex-cloud/bin/yc managed-kubernetes cluster get-credentials --external --name k8s-demo --force")
          sh("/var/lib/jenkins/kubectl get ns master || kubectl create ns master")
          sh("/var/lib/jenkins/kubectl apply -n master -f /var/lib/jenkins/deployment.yml")
          sh("/var/lib/jenkins/kubectl apply -n master -f /var/lib/jenkins/service.yml")
          echo 'Done!'
          
        }
    }
  }
}
```
###  Создание репозитория

```
https://github.com/tiesto-nk/demo
```

###  Создание pipeline

* Создаем Multibranch pipeline
![Создаем Multibranch pipeline](https://i.ibb.co/jwztC8H/11.png)

* Указываем в строке Branch source
![Branch source](https://i.ibb.co/NTMG7pQ/1.png)

* Указываем Jenkinsfile
![Jenkinsfile](https://i.ibb.co/WxZV3FX/2.png)

* Созраняем проект
После сохранения Jenkins сразу начнет сканировать репозиторий и начнет выполнять pipeline описаный Jenkinsfile

![3](https://i.ibb.co/tmrCd6W/3.png)

![4](https://i.ibb.co/8gkN2q5/4.png)

* Logs:

```
Branch indexing
09:54:37 Connecting to https://api.github.com with no credentials, anonymous access
09:54:37 GitHub API Usage: Current quota has 53 remaining (1 under budget). Next quota of 60 in 58 min
Obtained Jenkinsfile from 9be1eb16cca04f0f162ccef5e982bc057a7ce539
Running in Durability level: MAX_SURVIVABILITY
[Pipeline] Start of Pipeline
[Pipeline] node
Running on Jenkins in /var/lib/jenkins/workspace/11_master
[Pipeline] {
[Pipeline] stage
[Pipeline] { (Declarative: Checkout SCM)
[Pipeline] checkout
No credentials specified
Cloning the remote Git repository
Cloning with configured refspecs honoured and without tags
Cloning repository https://github.com/tiesto-nk/demo.git
 > git init /var/lib/jenkins/workspace/11_master # timeout=10
Fetching upstream changes from https://github.com/tiesto-nk/demo.git
 > git --version # timeout=10
 > git fetch --no-tags --progress -- https://github.com/tiesto-nk/demo.git +refs/heads/master:refs/remotes/origin/master # timeout=10
 > git config remote.origin.url https://github.com/tiesto-nk/demo.git # timeout=10
 > git config --add remote.origin.fetch +refs/heads/master:refs/remotes/origin/master # timeout=10
 > git config remote.origin.url https://github.com/tiesto-nk/demo.git # timeout=10
Fetching without tags
Fetching upstream changes from https://github.com/tiesto-nk/demo.git
 > git fetch --no-tags --progress -- https://github.com/tiesto-nk/demo.git +refs/heads/master:refs/remotes/origin/master # timeout=10
Checking out Revision 9be1eb16cca04f0f162ccef5e982bc057a7ce539 (master)
 > git config core.sparsecheckout # timeout=10
 > git checkout -f 9be1eb16cca04f0f162ccef5e982bc057a7ce539 # timeout=10
Commit message: "Update README.md"
First time build. Skipping changelog.
[Pipeline] }
[Pipeline] // stage
[Pipeline] withEnv
[Pipeline] {
[Pipeline] withEnv
[Pipeline] {
[Pipeline] stage
[Pipeline] { (Deploy demo)
[Pipeline] sh
+ export KUBECONFIG=/var/lib/jenkins/test.kubeconfig
[Pipeline] sh
+ /var/lib/jenkins/yandex-cloud/bin/yc container registry configure-docker
Credential helper is configured in '/var/lib/jenkins/.docker/config.json'
[Pipeline] sh
+ /var/lib/jenkins/yandex-cloud/bin/yc managed-kubernetes cluster get-credentials --external --name k8s-demo --force

Context 'yc-k8s-demo' was added as default to kubeconfig '/var/lib/jenkins/.kube/config'.
Check connection to cluster using 'kubectl cluster-info --kubeconfig /var/lib/jenkins/.kube/config'.

Note, that authentication depends on 'yc' and its config profile 'default'.
To access clusters using the Kubernetes API, please use Kubernetes Service Account.
[Pipeline] sh
+ /var/lib/jenkins/kubectl get ns master
NAME     STATUS   AGE
master   Active   69m
[Pipeline] sh
+ /var/lib/jenkins/kubectl apply -n master -f /var/lib/jenkins/deployment.yml
deployment.apps/demo-app unchanged
[Pipeline] sh
+ /var/lib/jenkins/kubectl apply -n master -f /var/lib/jenkins/service.yml
service/demo-service unchanged
[Pipeline] echo
Done!
[Pipeline] }
[Pipeline] // stage
[Pipeline] }
[Pipeline] // withEnv
[Pipeline] }
[Pipeline] // withEnv
[Pipeline] }
[Pipeline] // node
[Pipeline] End of Pipeline
Finished: SUCCESS
```

* Поднятые контейнеры

![5](https://i.ibb.co/ZNztt7P/5.png)


### Ссылка на приложение : http://130.193.49.227/

