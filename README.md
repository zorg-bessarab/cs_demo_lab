# Интро
__Это пошаговое руководство по развертыванию демо окружения для тестирования системы защиты контейнеров__
## Окружение
Для проверки основных функций систем защиты контейнеров предполагается развернуть следующие сервисы:
- Кластер Kubernetes, который будет эмулировать инфраструктуру
- Базовый реестр образов с docker v2 API
- Систему сборки и доставки приложений на базе Jenkins
- Репозиторий с тестовым проектом, который будет устанавливаться в кластере Kubernetes. Для простоты возьмём проект [Kubernetes examples](https://github.com/kubernetes/examples?tab=readme-ov-file)
- Репозиторий с тестовым проектом рантайм. Для демо возьмём пример команды cilium [star wars demo](https://github.com/cilium/star-wars-demo.git)
- ldap-сервер, в котором будут храниться пользователи системы
## Ограничения стенда
Для простоты развёртывания и очистки тестового окружения кластер Kubernetes будем разворачивать в локальной среде kind.
Для доступа к webui всех сервисов [ingress-ngix](https://kind.sigs.k8s.io/docs/user/ingress/#ingress-nginx)
Стенд необходимо разворачивать на физическом или виртуальном сервере со следующими параметрами:
- 8 ядер CPU
- 16GB RAM
- 100GB disk
- Данное руководство тестировалось на виртуальной машине QEMU KVM (proxmox 8.1.10) с гостевой ОС Ubuntu 22.04 LTS
## Разворачиваем кластер Kubernetes
После создания виртуальной машины в вашей среде виртуализации логиним в неё по ssh
__Важно:__ Не устанавливайте docker из snap пакетов Ubuntu.
- Устанавливаем docker по инструкции с сайта [docker.com](https://docs.docker.com/engine/install/ubuntu/)
- Устанавливаем средство создания тестовых кластеров Kubernetes
   ```sh
   [ $(uname -m) = x86_64 ] && curl -Lo ./kind https://kind.sigs.k8s.io/dl/v0.22.0/kind-$(uname)-amd64
   chmod +x ./kind
   sudo mv ./kind /usr/local/bin/kind
   kind version
   ```
- Разворачиваем кластер с необходимой конфигурацией
```sh
#!/bin/sh
# deploy_cluster_registry.sh

set -o errexit
# create registry container unless it already exists
reg_name='registry.demo'
reg_port='5001'

if [ "$(docker inspect -f '{{.State.Running}}' "${reg_name}" 2>/dev/null || true)" != 'true' ]; then
  docker run \
    -d --restart=always -p "127.0.0.1:${reg_port}:5000" --name "${reg_name}" \
    registry:2
fi

# Add alias to hosts
echo 'Editing hosts alias....'
if uname -r | grep -qi microsoft; then
  echo "Add 127.0.0.1 ${reg_name} to C:/Windows/system32/drivers/etc/hosts"
elif grep -qiE -v "127.0.0.1.* ${reg_name}\b" /etc/hosts; then
    echo "127.0.0.1 ${reg_name}" | sudo tee -a /etc/hosts
fi


# create a cluster with the local registry enabled in containerd
cat << EOF | kind create cluster --config=-
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
- role: control-plane
  kubeadmConfigPatches:
  - |
    kind: InitConfiguration
    nodeRegistration:
      kubeletExtraArgs:
        node-labels: "ingress-ready=true"
  extraPortMappings:
  - containerPort: 80
    hostPort: 80
    protocol: TCP
  - containerPort: 443
    hostPort: 443
    protocol: TCP
  extraMounts:
    - hostPath: /proc
      containerPath: /procHost
containerdConfigPatches:
- |-
  [plugins."io.containerd.grpc.v1.cri".registry]
    [plugins."io.containerd.grpc.v1.cri".registry.mirrors]
      [plugins."io.containerd.grpc.v1.cri".registry.mirrors."${reg_name}:${reg_port}"]
        endpoint = ["http://${reg_name}:5000"]
    [plugins."io.containerd.grpc.v1.cri".registry.configs]
      [plugins."io.containerd.grpc.v1.cri".registry.configs."${reg_name}:5000".tls]
        insecure_skip_verify = true
EOF

# connect the registry to the cluster network if not already connected
if [ "$(docker inspect -f='{{json .NetworkSettings.Networks.kind}}' "${reg_name}")" = 'null' ]; then
  docker network connect "kind" "${reg_name}"
fi

# Document the local registry
# https://github.com/kubernetes/enhancements/tree/master/keps/sig-cluster-lifecycle/generic/1755-communicating-a-local-registry
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: ConfigMap
metadata:
  name: local-registry-hosting
  namespace: kube-public
data:
  localRegistryHosting.v1: |
    host: "${reg_name}:${reg_port}"
    help: "https://kind.sigs.k8s.io/docs/user/local-registry/"
EOF
```
```sh
./deploy_cluster_registry.sh
```
_При необходимости конфигурацию можно изменить:
- В конфигурации настраиваем порты, на которых будет доступен ingress (по-умолчанию 443 и 80)
- Дополнительные маунты для доступа к хостовой ОС (необходимы для корректной работы в kind компонтов tetragon)
- Настраиваем возможность доверять самоподписанному сертификату реестра_
## Настройка кластера для стенда
1. Настраиваем ингресс для публикации сервисов
```sh
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/main/deploy/static/provider/kind/deploy.yaml

```
2. Устанавливаем Jenkins CI/CD из официального репозитория.
```yaml
# jenkins-values.yaml
# меняем пароль для администратора
controller:
  admin:
    createSecret: true
    username: "jenkins"
    password: "jenkins"
  # Плагин для запуска kubectl в агентах jenkins
  additionalPlugins:
  - kubernetes-cli:1.12.1
  - generic-webhook-trigger:2.2.1
# Публикация сервиса через ингресс
  ingress:
    enabled: true
    apiVersion: "networking.k8s.io/v1"
    ingressClassName: nginx
# Создание SA для агента
  serviceAccountAgent:
    create: true
    name: jenkins-agent
```

```sh
helm repo add jenkins https://charts.jenkins.io
helm install myjenkins jenkins/jenkins -f jenkins-values.yaml
```
3. Создаем тестовый проект пайплайна с возможностью интеграции деплоя в k8s.
Для работы из контейнера нам необходимо создать сервис аккаунт для агента, для простоты выдадим кластер админа. 
**В реальной среде создавать роль с минимальными полномочиями, которыми хотите пользоваться**
```sh
kubectl create serviceaccount jenkins-robot
kubectl create rolebinding jenkins-robot-binding --clusterrole=cluster-admin --serviceaccount=default:jenkins-robot
kubectl apply -f - <<EOF
apiVersion: v1
kind: Secret
metadata:
  name: jenkins-robot-secret
  annotations:
    kubernetes.io/service-account.name: jenkins-robot
type: kubernetes.io/service-account-token
EOF
# Забираем токен из секрета и передаём его в credentials в формате text secret
kubectl get secrets jenkins-robot-secret -o go-template --template '{{index .data "token"}}' | base64 -d
# После этого забираем id credential и вставляем в конфигурацию пайплайна
```
4. Создаём проект, который может управлять кластером через webhook```
```groovy
pipeline {
  agent any
    triggers {
    GenericTrigger(
     genericVariables: [
         [defaultValue: '', key: 'pod', regexpFilter: '', value: '$.events[0].pod_name'], 
         [defaultValue: '', key: 'node', regexpFilter: '', value: '$.events[0].node_name'],
         [defaultValue: '', key: 'event', regexpFilter: '', value: '$']
         ],

     causeString: 'Triggered on $event',

     token: '',
     tokenCredentialId: '',

     printContributedVariables: true,
     printPostContent: true,

     silentResponse: false,
     
     shouldNotFlatten: false,

     regexpFilterText: '',
     regexpFilterExpression: ''
    )
  }
  stages {
    stage('Scale replicas') {
        steps{
        withKubeConfig([credentialsId: '0f521642-12ee-4a9d-b6e2-deb81551aec2', serverUrl: 'https://kubernetes.default']) {
           sh 'curl -LO "https://storage.googleapis.com/kubernetes-release/release/v1.20.5/bin/linux/amd64/kubectl"'  
           sh 'chmod u+x ./kubectl'  
           sh """./kubectl scale --replicas=0 deployment \$(./kubectl get replicaset \$(./kubectl get pod $pod -o jsonpath='{.metadata.ownerReferences[0].name}') -o jsonpath='{.metadata.ownerReferences[0].name}')"""
        }
        }
    }
    stage('Echo $event') {
      steps {
        sh "echo $event"
      }
    }
  }
}
```
5. Создаём проект, сканирующий конфиги
```groovy
pipeline {
  environment {
    ptcsAddr = "https://reverse-proxy.ptcs.svc.cluster.local:9000"
    ptcsUser = "admin"
    ptcsPassword = "demo"
    imageName = "registry.demo:5000/quay.io/cilium/hubble-export-stdout:v1.0.3"
    report_format = "html"
    configRuleID = "f75ca39f-b88d-4b72-b154-04d39530ad50"
    configDir = "guestbook-go"
  }
  agent any
  stages {
    stage('scm') {
	    steps {
			git branch: 'master', url: 'https://github.com/kubernetes/examples.git'
	    }
	}
    stage('Get latest ptcs-cli') {
      steps{
        sh "curl -LO -k $ptcsAddr/file/ptcs-cli"
        sh "chmod +x ./ptcs-cli"
        sh "./ptcs-cli scan image --log-level='DEBUG' --login=${ptcsUser} --password=${ptcsPassword} --ptcsurl=${ptcsAddr} --report-format=${report_format} --report-output=report.${report_format} -r ${imageName}"
      }
    }
    stage('Scan Config'){
      steps {      
          sh """
            find . -type f -name Dockerfile -path "*/${configDir}/*" -exec sh -c \'./ptcs-cli scan dockerfile --login=${ptcsUser} --password=${ptcsPassword} \
            --enforced-rules=${configRuleID} \
            --ptcsurl=${ptcsAddr} --log-level=DEBUG --tls-skip-verify --report-output=report_deockerfile.${report_format} \
            --report-format=${report_format} \
            \$1\' -- {} ";"
            """
          sh """
          echo "=== Scanning config folders==="
          echo "" > output.yaml
          find . -type f -name '*.yaml' -path "*/${configDir}/*" -print -exec sh -c 'cat \$1 >> output.yaml && echo -e "\n---\n" >> output.yaml ' -- {} ";"
          echo "=== Scanning config with PT CS: ===" 
          cat output.yaml
          ./ptcs-cli scan kubernetes --login=${ptcsUser} --password=${ptcsPassword} \
            --enforced-rules=${configRuleID} \
            --ptcsurl=${ptcsAddr} --log-level=DEBUG --tls-skip-verify --report-output=report_kubernetes.${report_format} \
            --report-format=${report_format} \
            output.yaml"""
        }
    }
  }
    post {
      always {
        archiveArtifacts artifacts: 'report*'
    }
    }
}
```
1. Устанавливаем и настраиваем тестовые приложения внутри кластера.
Для тестирования функционала рантайма и admission controll необходимо развернуть тестовые приложения, на которых будут демонстрироваться сработки. 
Для установки приложений запустим:
```sh
git clone https://github.com/cilium/star-wars-demo.git
cd star-wars-demo/
kubectl create -f 01-deathstar.yaml -f 02-xwing.yaml
```
5. Устанавливаем и настраиваем log management (ELK) и 
## Установка и настройка CS
1. Скачиваем дистрибутив и разархивируем
```sh
wget -O ptcs-0.4.1366.tar https://storage.ptsecurity.com/f/5d3a838f17a94e269bf0/?dl=1
mkdir ptcs_installer
tar xvf ptcs-0.4.1366.tar -C ptcs_installer/
cd ptcs_installer/
```
2. Загружаем образы в созданный нами реестр _registry.demo_
```sh
IMAGE_REGISTRY_PASSWORD=test ./push_images.sh -u test -r registry.demo:5000 -i
```
3. Генерируем файл values.yaml для установки CS
```sh
IMAGE_REGISTRY_PASSWORD=test ./generate_values.sh -u test -r registry.demo:5000 --ingress-class nginx -a admin --dev
```
   После запуска команды необходимо ввести пароли администратора и пароли для доступа к БД и кешу. Скрипт сам сгененрирует файл values.yaml, положит его в текущую директорию и выведеть команду установки CS. Перед установкой нужно изменить значение flusURL в values.yaml на https://update.ptsecurity.com:
   
   ```sh
   helm upgrade --install ptcs /home/cs/ptcs_installer/ptcs-v0.0.0-release-0.4.tgz --namespace ptcs --create-namespace --values /home/cs/ptcs_installer/values.yaml
   ```

   1. Редактируем ingress для доступа по именам
```sh
kubectl edit ingress myjenkins -n default
# spec:
#   ingressClassName: nginx
#   rules:
#   - host: jenkins.demo
#     http:
#       paths:
#       - backend:
#           service:
#             name: myjenkins
#             port:
#               number: 8080
#         pathType: ImplementationSpecific
kubectl edit ingress reverse-proxy-ingress -n ptcs
# spec:
#   ingressClassName: nginx
#   rules:
#   - host: ptcs.demo
#     http:
#       paths:
#       - backend:
#           service:
#             name: reverse-proxy
#             port:
#               number: 8000
```
   5. Также нужно добавить эти alias в ваш DNS или /etc/hosts
   6. Проверяем доступы. Логинимся в консоль. Активируем лицензию.
   7. Настраиваем сервис уведомлений webhook для Jenkins. Указываем имя кубового сервиса http://10.96.32.166:8080 и путь /generic-webhook-trigger/invoke
   8. Настраиваем правила для сканирования конфигураций и рантайма. Правилу рантайма указываем существующий шаблон.
   9. запускаем star-wars-demo/03-pod-cmdline.sh копируем команду и проверяем работу
   10. Вручную запускаем пайплайн для конфигов. (Указываем Id правила в переменных)
