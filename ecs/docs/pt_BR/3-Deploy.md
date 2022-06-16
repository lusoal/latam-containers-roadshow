# LATAM Containers Roadshow - Workshop de Amazon ECS

[**< Voltar**](./2-Build.md)

## Capítulo 3 - Implantando a Aplicação

Dado que já temos nossos ambientes de infraestrutura no ar, vamos agora implantar os componentes da nossa aplicação `todo`. Essa aplicação é composta por dois componentes, `backend` e `frontend`. Os códigos fonte desses componentes estão disponíveis em `ecs/src`, e possuem seus respectivos `Dockerfiles` já construídos e armazenados nos mesmos diretórios.

Assim como fizemos no capítulo anterior, vamos continuar explorando o AWS Copilot mas agora na implantação dos nossos `Services`, que representam nossas cargas de trabalho em execução e podem ser de vários tipos (Request-Driven Web Service, Load Balanced Web Service, Backend Service ou Worker Service), e de `Add-Ons`, que são serviços complementares que nossos componentes utilizam, como armazenamento ou banco de dados.

1. Vamos começar configurando o nosso `Service` para o componente de Backend, que vai ser do tipo `Backend Service`, e vai ser criado a partir da imagem de container disponível em `backend/Dockefile`. Vamos usar o comando interativo:

```bash
copilot svc init
```

Nas perguntas, respoda:

 - Which service type best represents your service's architecture? **Backend Service**
 - What do you want to name this service? **backend**
 - Which Dockerfile would you like to use for backend? **backend/Dockerfile**

![Captura de tela com o resultado do comando 'copilot svc init'](../static/3.1-copilot_svc_init_backend.png)

2. Vamos explorar o arquivo criado em `src/copilot/backend/manifest.yml`, olhar as opções de configuração que temos disponível, usando o editor de arquivos do AWS Cloud9. Vamos modificar esse arquivo para adicionar uma configuração de validação de saúde (health-check):

![Imagem animada editando o arquivo de manifesto do serviço backend](../static/3.2-modify_backend_manifest.gif)

Inserir este trecho, logo depois de `port: 10000`:

```yaml
  healthcheck:
    command: ["CMD-SHELL", "curl -f http://localhost:10000/ishealthy || exit 1"]
    interval: 10s
    retries: 2
    timeout: 6s 
    start_period: 10s
```

3. Esse componente de backend precisa de uma base de dados Amazon DynamoDB para desempenhar sua função. Vamos usar do AWS Copilot para configurar esse serviço adicional de forma integrada. Para isso, vamos executar:

```bash
copilot storage init
```

Nas perguntas, responda:

 - What type of storage would you like to associate with backend? **DynamoDB**
 - What would you like to name this DynamoDB Table? **todotable**
 - What would you like to name the partition key of this DynamoDB? **TodoId**
 - What datatype is this key? **Number**
 - Would you like to add a sort key to this table? **N**

![Captura de tela com o resultado do comando 'copilot storage init'](../static/3.3-copilot_storage_init.png)

4. Esse comando gerou um diretório em `src/copilot/backend/addons`, e um AWS CloudFormation stack chamado `todotable.yaml` que define o banco de dados DynamoDB que criamos. Atualmente o AWS Copilot suporta Amazon S3 (objetos), Amazon DynamoDB (NoSQL) e Amazon Aurora (SQL) como add-ons, mas você pode criar seus próprios stacks de adicionar à essa estrutura para que o AWS Copilot coordene o seu provisionamento com o `Service`.

![Animação de tela com stack de addon para dynamodb](../static/3.4-copilot_addon_dir.gif)

5. Em seguida, podemos realizar a implantação do nosso componente `backend` como um `Service` no ambiente de desenvolvimento. Esse processo vai gerar a imagem de container do nosso componente, enviar para o Amazon Elastic Container Registry (ECR), e depois provisionar os recursos necessários para operacionalizar o componente no Amazon ECS. Para isso, vamos executar:

```bash
copilot svc deploy
```

Na pergunta, responda:

 - Select an environment: **development**

![Captura de tela com o resultado do comando 'copilot svc deploy'](../static/3.5-copilot_svc_deploy_backend_dev.png)

6. Podemos usar o AWS Copilot para interagir com o serviço e validar o estado atual:

```bash
copilot svc status --name backend --env development
```

![Captura de tela com o resultado do comando 'copilot svc status'](../static/3.6-copilot_svc_status_backend_dev.png)

7. Agora vamos configurar o nosso componente de `frontend`, que será do tipo `Load Balanced Web Service`. Dessa vez vamos usar o comando completo ao invés do modo interativo:

```bash
copilot svc init -n frontend -t "Load Balanced Web Service" -d frontend/Dockerfile --port 80
```

![Captura de tela com o resultado do comando 'copilot svc init'](../static/3.7-copilot_svc_init_frontend.png)

8. E vamos implantá-lo também no nosso ambiente de desenvolvimento:

```bash
copilot svc deploy --app todo --env development --name frontend
```

![Captura de tela com o resultado do comando 'copilot svc deploy'](../static/3.8-copilot_svc_deploy_frontend_dev.png)

9. Diferente do serviço do tipo `Backend Service`, que não gera nenhum Elastic Load Balancer (ELB) para acesso externo, o do tipo `Load Balanced Web Service` cria um Application Load Balancer (ALB) público. Podemos testar nossa aplicação `todo` no ambiente de desenvolvimento!

![Animação de tela acessando a aplicação em desenvolvimento](../static/3.9-frontend_demo.gif)

[**Próximo >**](./4-Observe.md)