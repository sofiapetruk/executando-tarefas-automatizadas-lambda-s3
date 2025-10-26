# executando-tarefas-automatizadas-lambda-s3

## Automatizar a configuração do S3 Object Lambda com um modelo do CloudFormation
  O objetivo principal aqui é usar a Infraestrutura como Código (IaC), representada pelo CloudFormation,
  para provisionar e gerenciar de forma repetível e segura todos os recursos necessários para o S3 Object Lambda, 
  que são.

- **Ponto de Acesso (Access Point):** O ponto de entrada para as requisições, que aponta para o Object Lambda Access Point.
- **Ponto de Acesso do Object Lambda (Object Lambda Access Point):** O recurso que intercepta as requisições do S3 e as encaminha
para a função Lambda.
- **Função AWS Lambda:** O código que irá processar e modificar o objeto original.
- **Política de Acesso (IAM Role/Policy):** As permissões necessárias para que a função Lambda leia o objeto original do S3 e para
que o S3 Objeto Lambda Access Point possa invocar a Lambda.
- **Bucket S3 de Suporte:** O bucket original onde os dados estão armazenados (que não é acessado diretamente, mas sim através do
Object Lambda Access Point).

### Teoria do CloudFormation:
#### Recursos Necessários no CloudFormation:
- **AWS::S3::AccessPoint:** O Access Point original que aponta para o bucket subjacente.
- **AWS::S3::AccessPoint:** O Access Point do Object Lambda.
- **AWS::S3ObjectLambda::AccessPointPolicy:** A política que permite que o Object Lambda Access Point seja usado.
- **AWS::Lambda::Function:** A função Lambda.
- **AWS::IAM::Role:** A Função IAM para a Lambda e o Object Lambda, garantindo que a Lambda tenha permissão para obter o objeto
original (via s3:GetObject) e escrever o resultado para o Object Lambda.
- **AWS::S3ObjectLambda::AccessPointPolicy:** Uma política que permite que a função Lambda seja invocada pelo Object Lambda Access Point.

## Tarefas automatizadas com Amazon S3 e Lambda
A combinação S3 + Lambda é o padrão para arquiteturas serverless orientadas a eventos.

### Teoria na AWS:
- **S3 Event Notifications:** É o mecanismo clássico onde o S3 pode ser configurado para invocar uma função Lambda
(ou enviar uma mensagem para SNS/SQS) em resposta a eventos específicos (e.g., s3:ObjectCreated:*, s3:ObjectRemoved:*).
- **Diferença para Object Lambda:** A Event Notification é acionada depois da operação (PUT, POST, DELETE). O S3 Object
Lambda, por outro lado, intercepta a requisição GET para modificar o objeto em tempo real antes que ele seja retornado
ao solicitante.
- **AWS Lambda Execution Role (Função de Execução):** A função Lambda deve possuir uma IAM Role que lhe conceda as
permissões mínimas necessárias. Para a integração com S3, isso inclui:
    - s3:GetObject no Original Access Point ou Bucket.
    - logs:CreateLogGroup, logs:CreateLogStream, logs:PutLogEvents (para logs do CloudWatch).

## Entendendo o Amazon S3
- **Durabilidade e Escalabilidade:** O S3 é projetado para durabilidade de $99.999999999\%$ dos objetos. É um serviço de
armazenamento de objetos altamente escalável.
- **Recursos Chave para Object Lambda:**
    - **Buckets:** O contêiner de armazenamento principal.
    -  **Access Points:** Sub-recursos de um bucket que simplificam o gerenciamento de acesso. Eles são usados como o alvo para o Object Lambda.
    -  **S3 Object Lambda:** Uma capacidade que permite adicionar seu próprio código para processar uma requisição GET de um objeto S3 padrão,
      retornando um resultado transformado. Ele não altera o objeto original no S3.

## 4. Entendendo o AWS Lambda
### Teoria na AWS:
- **Modelo Serverless:** O Lambda executa seu código em resposta a eventos e gerencia automaticamente os recursos de computação subjacentes.
Você paga apenas pelo tempo de computação consumido.
- **Função do Manipulador (Handler):** O Object Lambda passa um evento especial para a Lambda. Este evento inclui o inputS3Url, uma URL
pré-assinada que a função usa para fazer uma requisição GET e obter o objeto original do S3.
- **Estrutura de Transformação:**
  1. O cliente solicita o objeto através do Object Lambda Access Point.
  2. O Object Lambda Access Point invoca a função Lambda.
  3. A função Lambda usa o inputS3Url para baixar o objeto original.
  4. A função processa o objeto (e.g., remove PII, compacta, muda o formato, etc.).
  5. A função Lambda usa a write-get-object-response API (via SDK) e o outputRoute e outputToken fornecidos no evento para escrever o objeto transformado
  de volta para o S3 Object Lambda, que então o retorna ao cliente.

## 5. Upload de Arquivos com Processamento e Registro no DynamoDB
Essa etapa combina o S3, Lambda e DynamoDB para criar um fluxo de trabalho.
### Teoria na AWS:
- **DynamoDB:** Banco de dados NoSQL de chave-valor e documentos totalmente gerenciado, conhecido por sua baixa latência em qualquer escala.
-  **Fluxo de Evento (Típico, não Object Lambda):**
    1. Um arquivo é enviado para o S3 (evento s3:ObjectCreated:Put).
    2. O S3 dispara uma Event Notification para a Função Lambda de Processamento.
    3. A Função Lambda de Processamento:
       - Lê o novo objeto do S3.
       - Processa o conteúdo (e.g., extrai metadados, valida dados).
       - Cria um novo registro no DynamoDB (e.g., nome do arquivo, carimbo de data/hora, status de processamento).
- **Recursos CloudFormation:** Adicione AWS::DynamoDB::Table e atualize a IAM Role da Lambda para incluir permissões (dynamodb:PutItem, dynamodb:UpdateItem, etc.) na tabela.

## Configuração AWS localmente com LocalStack
- **LocalStack**: É uma estrutura que simula localmente a infraestrutura de serviços da AWS (S3, Lambda, DynamoDB, CloudFormation, etc.). Isso permite o desenvolvimento e
testes offline ou em um ambiente de CI/CD local, reduzindo a dependência do ambiente de nuvem real e custos.

**Vantagens:**
- Desenvolvimento Rápido (Fast Feedback Loop).
- Teste de Integração Local.
- Economia de Custos.

**Como Usar (Conceito):** Você usa a CLI da AWS (awscli) ou SDKs, mas aponta o endpoint para o LocalStack (e.g., --endpoint-url=http://localhost:4566). Você pode implantar 
seu template do CloudFormation no ambiente LocalStack para simular o provisionamento.

## 7. Criando os recursos e 8. Trabalhando arquviso localmente com LocalStack
**Resumo Teórico do Fluxo de Trabalho (IaC + LocalStack):**

1. **Modelo CloudFormation:** Defina todos os recursos (S3 Object Lambda Access Point, Lambda, DynamoDB, IAM Roles) em um arquivo YAML/JSON (template.yaml).

2. **Preparação da Lambda:** Empacote o código da sua função Lambda (que usa o SDK para o fluxo de Object Lambda ou DynamoDB).

3. **Implantação Local:** Use o aws cloudformation deploy (ou ferramentas como AWS SAM ou Terraform) apontando para o endpoint do LocalStack para provisionar todos os recursos simulados.

4. **Teste Local:**
- **S3 Object Lambda:** Use a CLI para fazer um s3:GetObject no Object Lambda Access Point simulado no LocalStack.
- **Resultado Esperado:** O LocalStack irá invocar sua Lambda localmente, a Lambda fará o processamento (simulando a leitura do objeto original) e o resultado transformado será retornado, confirmando que toda a cadeia de Object Lambda está funcional.
