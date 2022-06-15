# LATAM Containers Roadshow - Workshop de Amazon ECS

[**< Voltar**](./1-Prepare.md)

## Capítulo 2 - Construindo a Infraestrutura

Agora que temos tudo o que precisamos em nosso ambiente de trabalho, neste capítulo vamos construir a infraestrutura que hospedará nossa aplicação em containers. Este workshop explorará como aproveitar o Amazon Elastic Compute Service (ECS) para operar contêineres em escala, aproveitando o vasto ecossistema de serviços AWS, sem comprometer a simplicidade operacional do ambiente.

Existem várias maneiras de provisionar infraestrutura na AWS, como usar o AWS CloudFormation, o AWS Cloud Development Kit (CDK) ou o Hashicorp Terraform. Nesse workshop vamos usar o AWS Copilot, que simplifica a criação e operação de aplicações containerizadas na AWS, incorporando as boas práticas para desenvolvimento de arquiteturas nativas de nuvem.

1. Vamos inicialmente clonar o respositório base do workshop para reaproveitar alguns dos artefatos e usar como espaço para inicialização do AWS Copilot.

```bash
cd ~/environment/
git clone https://github.com/davivcgarcia/latam-containers-roadshow.git
cd latam-containers-roadshow/ecs/src/
```

2. A AWS Copilot é uma ferramenta de linha de comando, e já se encontra disponível no nosso ambiente de trabalho do AWS Cloud9. Vamos explorar quais são as opções disponíveis. Você pode usar desse comando como subcomandos para explorar o que cada uma das opções aceita de parâmetros.

```bash
copilot help
```

![Captura de tela com as opções de comandos disponíveis do AWS Copilot](../static/2.1-copilot_options.png "2.1 - Opções de comandos no AWS Copilot")

3. Vamos seguir a recomendação do comando anterior, e inicializar nossa aplicação com o AWS Copilot. Para isso, vamos executar o comando abaixo:

```bash
copilot app init todo
```

![Captura de tela com o resultado do comando 'app init'](../static/2.2-copilot_app_init.png "2.2 - Saída do comando 'copilot app init'")

4. Esse comando criou recursos como IAM Role e parâmetros no AWS System Manager Parameter Store através do AWS CloudFormation que permitem a ferramenta gerenciar outros componentes como `Services` e `Jobs`. Podemos listar as aplicações AWS Copilot com:

```bash
copilot app ls
```

![Captura de tela com o resultado do comando 'app ls'](../static/2.3-copilot_app_ls.png "2.3 - Saída do comando 'copilot app ls'")

5. Além disso, ele criou uma estrutura de diretórios a partir de onde o comando de inicialização foi executado, onde ele armazena os arquivos de configuração dos componentes relacionados ao AWS Copilot. Esse diretório ainda está praticamente vazio, mas ele vai ficar populado em breve!

```bash
tree -a .
```

![Captura de tela com o resultado do comando 'tree -a'](../static/2.4-copilot_dir.png "2.4 - Saída do comando 'tree -a'")

6. Uma vez que a nossa `Application` foi criada, vamos criar os `Environments` no AWS Copilot. Estes são de fato as infraestruturas que suportam as nossas `Applications` e, que no nosso caso, será baseada em Amazon Elastic Container Service (ECS) e AWS Fargate. Vamos primeiro criar o nosso ambiente de desenvolvimento, e siga com as opções padrão para as perguntas:

```bash
copilot env init --app todo --name development --container-insights
```

![Captura de tela com o resultado do comando 'copilot env init'](../static/2.5-copilot_env_init.png "2.5 - Saída do comando 'copilot env init'")

7. Perceba que o AWS Copilot uma série de recursos, dentre eles uma Virtual Private Cloud (VPC), IAM Roles e o nosso cluster de Amazon ECS. O AWS Copilot usa o AWS Fargate como motor de execução dos containers, trazendo uma abordagem serverless (não há instâncias de Amazon EC2 a serem gerenciadas!). Toda essa configuração de recursos se dá através do Amazon CloudFormation.

![Imagem animada com as saídas do AWS CloudFormation](../static/2.6-stacks_cloudformation.gif "2.6 - Captura de tela ")

8. Vamos repetir o processo para criar nosso ambiente produtivo. Poderíamos estar usando contas AWS diferentes, mas por conta da limitação do laboratórios vamos ter ambos ambientes na mesma conta mas em Virtual Private Clouds (VPCs) diferentes. Dessa vez vamos usar a opção `--default-config` para ele não perguntar nada sobre a infraestrutura e seguir com o padrão.

```bash
copilot env init --app todo --name production --container-insights --default-config
```

...

[**Próximo >**](./3-Deploy.md)