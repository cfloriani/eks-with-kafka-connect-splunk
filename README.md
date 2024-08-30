
# Kafka Connect com EKS

Essa documentação parte do principio que o você já possui um ambiente Kafka e Splunk implantado e funcional. Iremos abordar a implantação do Kafka-connect e suas configurações para implantar o conector do Splunk e realizar o envio das mensagens armazenadas nos tópicos do Kafka para o Splunk.

![Diagrama](https://cdn.splunkbase.splunk.com/media/public/screenshots/66764b4c-473c-11e8-968a-026b1e18168c.png)
Aplicação > Tópico < Splunk Connect for Kafka > Splunk

## 1. Configuração do Ambiente EKS
### 1.1 Criar Cluster EKS

Pré-requisitos:
* AWS CLI instalado e configurado.
* `kubectl` instalado.
* `eksctl` instalado.
* `helm` instalado.
* Kafka configurado e funcional
* Splunk Enterprise|Cloud com HEC ativado
```
eksctl create cluster \
--name kafka-connect-cluster \
--version 1.24 \
--region us-east-1 \
--nodegroup-name standard-workers \
--node-type t3.medium \
--nodes 3 \
--nodes-min 1 \
--nodes-max 4 \
--managed
```
### 1.2 Configurar o kubectl para usar o novo cluster
```
aws eks --region us-east-1 update-kubeconfig --name kafka-connect-cluster
```
### 1.3 Verificar se o cluster está funcionando corretamente
```
kubectl get nodes
```
## 2. Configuração do Kafka Connect no EKS
### 2.1 Implantar Zookeeper e Kafka no EKS
Utilize Helm para implantar o Zookeeper e o Kafka:
```
helm repo add bitnami https://charts.bitnami.com/bitnami
helm install zookeeper bitnami/zookeeper --namespace default
helm install kafka bitnami/kafka --namespace default
```
### 2.2 Implantar Kafka Connect no EKS
Crie um arquivo kafka-connect-deployment.yaml com a configuração básica do Kafka Connect:
```
apiVersion: apps/v
kind: Deployment
metadata:
name: kafka-connect
namespace: default
spec:
replicas: 1
selector:
matchLabels:
app: kafka-connect
template:
metadata:
labels:
app: kafka-connect
spec:
containers:
- name: kafka-connect
image: confluentinc/cp-kafka-connect:latest
ports:
- containerPort: 8083
env:
- name: CONNECT_BOOTSTRAP_SERVERS
value: "kafka:9092"
- name: CONNECT_REST_PORT
value: "8083"
- name: CONNECT_GROUP_ID
value: "kafka-connect-group"
- name: CONNECT_CONFIG_STORAGE_TOPIC
value: "kafka-connect-configs"
- name: CONNECT_OFFSET_STORAGE_TOPIC
value: "kafka-connect-offsets"
- name: CONNECT_STATUS_STORAGE_TOPIC
value: "kafka-connect-status"
- name: CONNECT_KEY_CONVERTER
value: "org.apache.kafka.connect.storage.StringConverter"
- name: CONNECT_VALUE_CONVERTER
value: "org.apache.kafka.connect.storage.StringConverter"
- name: CONNECT_PLUGIN_PATH
value: "/usr/share/java,/etc/kafka-connect/jars"
volumeMounts:
- name: kafka-connect-plugins
mountPath: /etc/kafka-connect/jars
volumes:
- name: kafka-connect-plugins
emptyDir: {}
```
Implante o Kafka Connect:
```
kubectl apply -f kafka-connect-deployment.yaml
```
## 3. Compilação do Conector Kafka Connect for Splunk
### 3.1 Clonar o Repositório do Conector
```
git clone https://github.com/splunk/kafka-connect-splunk.git cd kafka-connect-splunk
```
### 3.2 Compilar o Conector
Certifique-se de que o Maven está instalado. Execute o comando de compilação:
```
mvn clean package
```
Após a compilação, o JAR será gerado no diretório target.

## 4. Inclusão do Conector Compilado no Kafka Connect
### 4.1 Copiar o JAR para o Kafka Connect
Faça o upload do arquivo JAR compilado para um local acessível no cluster EKS, por exemplo, usando um serviço de armazenamento como o S3.
Em seguida, modifique o kafka-connect-deployment.yaml para montar esse JAR:
```
volumes:

- name: kafka-connect-plugins
emptyDir: {}
# Adicione o seguinte se estiver usando o S3 para baixar o JAR
initContainers:
- name: init-kafka-connect-plugins
image: amazonlinux
command: ['sh', '-c', 'aws s3 cp s3://your-bucket-name/target/splunk-
kafka-connect-<version>.jar /etc/kafka-connect/jars/']
volumeMounts:
- name: kafka-connect-plugins
mountPath: /etc/kafka-connect/jars
```
Reaplique o arquivo kafka-connect-deployment.yaml:
```
kubectl apply -f kafka-connect-deployment.yaml
```
## 5. Configuração do Conector para Enviar Dados ao Splunk
### 5.1 Criar o Conector Splunk HEC no Kafka Connect
Use um comando curl para criar e configurar o conector Splunk HEC:
```
curl -X POST -H "Content-Type: application/json" --data '{
"name": "splunk-sink-connector",
"config": {
"connector.class": "com.splunk.kafka.connect.SplunkSinkConnector",
"tasks.max": "1",
"topics": "firewall",
"splunk.hec.uri": "https://your-splunk-hec-endpoint:8088",
"splunk.hec.token": "your-hec-token",
"splunk.hec.ack.enabled": "true",
"splunk.hec.raw": "false",
"splunk.hec.source": "kafka",
"splunk.hec.sourcetype": "_json",
"splunk.hec.index": "main",
"key.converter": "org.apache.kafka.connect.storage.StringConverter",
"value.converter": "org.apache.kafka.connect.storage.StringConverter"
}
}' http://<kafka-connect-service>:8083/connectors
```
### 5.2 Verificação
Verifique se o conector está ativo e enviando dados para o Splunk realizando buscas no index utilizado.


