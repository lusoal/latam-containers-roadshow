# LATAM Containers Roadshow - ECS Workshop

[**< Back**](./3-Deploy.md)

## Chapter 4 - Explore Observability

7. Uma vez o `Service` em execução, podemos obter os logs do nosso componente de `backend` com:

```bash
copilot svc logs --name backend --env development
```

![Captura de tela com o resultado do comando 'copilot svc status'](../static/3.7-copilot_svc_logs.png)

8. Também podemos usar do AWS Copilot para acessar o container em execução dentro do ambiente Amazon ECS, mesmo usando AWS Fargate como motor de execução dos containers. Esse tipo de acesso pode ser importante para eventuais resoluções de problemas ou depuração:

```bash
copilot svc exec --name backend --env development
```

![Captura de tela com o resultado do comando 'copilot svc exec'](../static/3.8-copilot_svc_exec.png)

[**Next >**](./5-Automate.md)