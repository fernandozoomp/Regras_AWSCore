# Regras_AWSCore

## Autorizar chamadas diretas para serviços da AWS

Os dispositivos podem usar certificados X.509 para se conectar ao AWS IoT Core usando protocolos de
autenticação mútua TLS. Outros serviços da AWS não são compatíveis com autenticação realizada por
meio de certificado, mas podem ser chamados usando credenciais da AWS no formato do AWS Signature
versão 4. <BR>
O algoritmo do Signature versão 4 normalmente exige que o chamador tenha um ID de chave
de acesso e uma chave de acesso secreta. 
O AWS IoT Core tem um provedor de credenciais que permite
usar o certificado X.509 integrado como o dispositivo exclusivo de identidade para autenticar solicitações
da AWS. Isso elimina a necessidade de armazenar um ID de chave de acesso e uma chave de acesso
secreta em seu dispositivo. <BR>
O provedor de credenciais autentica um chamador usando um certificado X.509 e emite um token de
segurança temporário e de privilégio limitado. Esse token pode ser usado para assinar e autenticar
qualquer solicitação da AWS. <BR>
Essa forma de autenticar solicitações da AWS requer que você crie
e configure uma função do AWS Identity and Access Management (IAM) e anexe políticas do IAM
adequadas à função para que o provedor de credenciais possa assumir a função em seu nome. Para obter
mais informações sobre AWS IoT Core e IAM, consulte Identity and Access Management para o AWS
IoT. <BR>
O diagrama a seguir mostra o fluxo de trabalho do provedor de credenciais.

<div align="center">

![Key](https://raw.githubusercontent.com/fernandozoomp/Regras_AWSCore/main/Data_AWS/Key_AWS.png)

</div>
 
## Criando regras para rotear dados dos dispositivos conectados

### As regras são compostas de:

#### Nome da regra

   Pode ser definido um padrão para facilitar o entendimento da regra.

#### Declaração do SQL

   Uma sintaxe SQL simplificada para filtrar as mensagens recebidas em um tópico do MQTT e enviar os
dados para outro lugar.

Para criar uma regra (AWS CLI) use o comando ``` create-topic-rule ``` para criar uma regra:

```
aws iot create-topic-rule --rule-name my-rule --topic-rule-payload file://my-rule.json
```

A seguir temos um exemplo de arquivo de carga com uma regra que insere todas as mensagens
enviadas para o tópico iot/test em uma tabela do DynamoDB especificada. A declaração do SQL filtra
as mensagens, e o ARN da função concede à AWS IoT permissão para gravar na tabela do DynamoDB.

```
{
 "sql": "SELECT * FROM 'iot/test'",
 "ruleDisabled": false,
 "awsIotSqlVersion": "2016-03-23",
 "actions": [{
 "dynamoDB": {
 "tableName": "my-dynamodb-table",
 "roleArn": "arn:aws:iam::123456789012:role/my-iot-role",
 "hashKeyField": "topic",
 "hashKeyValue": "${topic(2)}",
 "rangeKeyField": "timestamp",
 "rangeKeyValue": "${timestamp()}"
 }
 }]
}
```
A seguir temos um exemplo de arquivo de carga com uma regra que insere todas as mensagens
enviadas para o tópico iot/test no bucket do S3 especificado. A declaração do SQL filtra as
mensagens, e o ARN da função concede à AWS IoT permissão para gravar no bucket do Amazon S3.

```
{
 "awsIotSqlVersion": "2016-03-23",
 "sql": "SELECT * FROM 'iot/test'",
 "ruleDisabled": false,
 "actions": [
 {
 "s3": {
 "roleArn": "arn:aws:iam::123456789012:role/aws_iot_s3",
 "bucketName": "my-bucket",
 "key": "myS3Key"
 }
 }
 ]
} 
```
Veja a seguir um exemplo de arquivo de carga com uma regra que envia dados ao Amazon
Elasticsearch Service:
```
{
 "sql":"SELECT *, timestamp() as timestamp FROM 'iot/test'",
 "ruleDisabled":false,
 "awsIotSqlVersion": "2016-03-23",
 "actions":[
 {
 "elasticsearch":{
 "roleArn":"arn:aws:iam::123456789012:role/aws_iot_es",
"endpoint":"https://my-endpoint",
 "index":"my-index",
 "type":"my-type",
 "id":"${newuuid()}"
 }
 }
 ]
}
```
Veja a seguir um exemplo de arquivo de carga útil com uma regra que invoca uma função do Lambda:
```
{
 "sql": "expression",
 "ruleDisabled": false,
 "awsIotSqlVersion": "2016-03-23",
 "actions": [{
 "lambda": {
 "functionArn": "arn:aws:lambda:us-west-2:123456789012:function:my-lambdafunction"
 }
 }]
}
```
Vejamos a seguir um exemplo de arquivo de carga útil com uma regra que publica novamente em outro tópico
do MQTT:
```
{
 "sql": "expression",
 "ruleDisabled": false,
 "awsIotSqlVersion": "2016-03-23",
 "actions": [{
 "republish": {
 "topic": "my-mqtt-topic",
 "roleArn": "arn:aws:iam::123456789012:role/my-iot-role"
 }
 }]
}
```
Veja a seguir um exemplo de arquivo de carga útil com uma regra que usa a função Amazon Machine
Learning do machinelearning_predict para publicar novamente em um tópico caso os dados na
carga útil do MQTT estejam classificados como 1.
```
{
 "sql": "SELECT * FROM 'iot/test' where machinelearning_predict('my-model',
 'arn:aws:iam::123456789012:role/my-iot-aml-role', *).predictedLabel=1",
 "ruleDisabled": false,
 "awsIotSqlVersion": "2016-03-23",
 "actions": [{
 "republish": {
 "roleArn": "arn:aws:iam::123456789012:role/my-iot-role",
 "topic": "my-mqtt-topic"
 }
 }]
}
```

### Visualizar as regras

Use o comando ```list-topic-rules``` para ver as regras:
```
aws iot list-topic-rules
```

Use o comando ```get-topic-rule``` para obter informações sobre uma regra:
```
aws iot get-topic-rule --rule-name my-rule
```
### Excluir uma regra

Quando não precisar mais de uma regra, você poderá excluí-la.
Para excluir uma regra (AWS CLI)
Use o comando ```delete-topic-rule``` para excluir uma regra:
```
aws iot delete-topic-rule --rule-name my-rule
```

