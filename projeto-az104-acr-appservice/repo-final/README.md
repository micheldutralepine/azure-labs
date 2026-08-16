# Deploy de Container no Azure App Service com CI/CD via ACR

Projeto de estudo prático para a certificação **AZ-104 (Microsoft Azure Administrator)**, cobrindo o ciclo completo de containerização e deploy contínuo no Azure: build de imagem Docker, push para o Azure Container Registry (ACR), deploy em Azure App Service e configuração de Continuous Deployment via webhook.

## Sobre este projeto

Este repositório documenta a aplicação prática de um conteúdo teórico sobre serviços de container no Azure (ACR, App Service, Container Instances, AKS, Red Hat OpenShift e Container Apps), usando um site HTML simples como carga de trabalho para validar o fluxo de ponta a ponta.

O site utilizado como conteúdo (`index.html` e páginas relacionadas) é material de treinamento de terceiros, distribuído sob licença Apache 2.0. Ver seção [Créditos](#créditos) abaixo. O foco deste repositório não é o conteúdo do site em si, mas a infraestrutura e o pipeline de deploy construídos ao redor dele.

## Arquitetura

```
Docker (local)  --build-->  imagem site-html
       |
       v
   docker push
       |
       v
Azure Container Registry (ACR)
       |
       | webhook (push event)
       v
Azure App Service  --pull-->  container em execução (nginx:alpine)
       |
       v
   site publicado
```

## Stack utilizada

- **Docker** — build da imagem local
- **Azure Container Registry (ACR)** — armazenamento privado da imagem
- **Azure App Service (Linux, container)** — hospedagem do container
- **nginx:alpine** — imagem base do container
- **Webhook do ACR** — gatilho de Continuous Deployment

## Dockerfile

```dockerfile
FROM nginx:alpine
COPY . /usr/share/nginx/html
```

Simples por design: a imagem base do nginx serve os arquivos estáticos do site copiados para o diretório padrão do servidor web.

## Passo a passo do processo

### 1. Build e push da imagem

```bash
az login
docker build -t site-html .
az acr login --name containerACR
docker tag site-html containeracr.azurecr.io/site-html
docker push containeracr.azurecr.io/site-html
```

Comandos completos em [`container.azcli`](./container.azcli).

Antes do push, o container foi validado rodando localmente via Docker Desktop (porta `8080:80`), confirmando que a imagem servia o conteúdo corretamente antes de subir para o Azure:
![Container rodando localmente no Docker Desktop](./docs/screenshots/00-docker-desktop-local-container.png)

### 2. Criação do Web App conectado ao ACR

O App Service foi configurado na aba **Container** durante a criação, apontando diretamente para a imagem hospedada no ACR.

Duas formas de autenticação foram testadas:

| Método | Comportamento |
|---|---|
| **Managed Identity** | Mais seguro, mas exige permissão `AcrPull` configurada manualmente e preenchimento manual dos campos Image/Tag (não vêm com autocomplete) |
| **Admin Credentials** | Mais simples para ambiente de estudo — os campos Image e Tag vêm com dropdown auto-populado a partir do registry |

![Managed Identity](./docs/screenshots/01-create-webapp-managed-identity.png)
![Admin Credentials](./docs/screenshots/02-create-webapp-admin-credentials.png)

Para este projeto, optou-se por **Admin Credentials**, adequado para ambiente de aprendizado.

### 3. Resumo da configuração antes da criação

![Review + Create](./docs/screenshots/03-review-create-summary.png)

### 4. Deploy concluído

Com o recurso criado, o App Service ficou em estado `Running`, com a imagem `containeracr.azurecr.io/site-html:latest` puxada corretamente do ACR.

![App Service Overview](./docs/screenshots/04-app-service-overview-running.png)
![Site no ar](./docs/screenshots/05-site-live-first-version.png)

### 5. Configuração de Continuous Deployment (webhook)

Para que o App Service atualize automaticamente sempre que uma nova imagem for enviada ao ACR, foi configurado um webhook no Container Registry apontando para o endpoint de deploy (SCM) do App Service.

Após a alteração do conteúdo do site e um novo `docker push`, o webhook disparou e o App Service atualizou automaticamente, sem intervenção manual:

![Site atualizado via CD](./docs/screenshots/06-site-updated-continuous-deployment.png)

## Troubleshooting real enfrentado

Documentar os erros e suas soluções é a parte mais valiosa deste projeto — reflete o tipo de diagnóstico exigido na prática (e na certificação AZ-104).

### Erro 1 — Quota de VMs insuficiente

**Sintoma:**
```
Operation cannot be completed without additional quota.
Current Limit (Total VMs): 0
Current Usage: 0
Amount required for this deployment (Total VMs): 1
```

**Causa:** a subscription de estudos tinha cota zerada para VMs em várias regiões (East US, Brazil South), comum em contas trial/estudante recém-criadas.

**Solução:** testar a criação do recurso em regiões alternativas. `Canada Central` tinha cota disponível para o SKU Free (F1), o que resolveu o bloqueio sem necessidade de solicitar aumento de cota via suporte.

### Erro 2 — Webhook do ACR retornando HTTP 401

Após configurar o Continuous Deployment, o histórico de eventos do webhook mostrava falha de autenticação:

![Erro 401 no webhook](./docs/screenshots/07-webhook-401-error.png)

**Diagnóstico:**

1. Confirmado que o payload do evento de `push` estava sendo gerado corretamente pelo ACR (validado via JSON do evento)
2. Confirmado que a opção **SCM Basic Auth Publishing Credentials** estava habilitada no App Service (`Settings > Configuration > General settings`), pré-requisito para o webhook autenticar via usuário/senha:

![SCM Basic Auth habilitado](./docs/screenshots/08-scm-basic-auth-setting.png)

3. Com a configuração correta e o erro persistindo, a causa raiz identificada foi: a **senha nas credenciais de publicação (deployment credentials) estava desatualizada** em relação à URL usada no webhook.

**Solução:**
1. `Reset publish profile` no App Service, gerando novas credenciais
2. Nova cópia da Webhook URL (formato `https://$app:senha@app.scm.region.azurewebsites.net/api/registry/webhook`) a partir do Deployment Center
3. Atualização do campo `Service URI` do webhook no ACR com a nova URL
4. Teste via botão **Ping**, retornando **202**:

![Ping com sucesso](./docs/screenshots/09-webhook-ping-success-202.png)

**Nota de segurança:** como a senha foi exposta durante o processo de debug, o publish profile foi resetado novamente após a resolução do problema, invalidando a credencial anteriormente exposta.

### Validação final

Novo ciclo completo de build → push → webhook → deploy automático, confirmando o CI/CD funcionando de ponta a ponta:

![Site atualizado - versão final](./docs/screenshots/10-site-updated-v2-final.png)

## Principais aprendizados

- Diferença prática entre autenticação via **Managed Identity** e **Admin Credentials** ao conectar App Service a um ACR
- Cotas de subscription (`vCPU quota`) são um bloqueador comum em contas de estudo e variam por região — vale testar regiões alternativas antes de abrir chamado de suporte
- O **SCM Basic Auth** é pré-requisito para webhooks de registry autenticarem via usuário/senha no App Service
- Credenciais de publicação (deployment credentials) podem ficar desatualizadas e precisam ser resetadas quando a autenticação falha mesmo com a configuração aparentemente correta
- O payload do evento do webhook (visível em JSON no portal) é uma ferramenta útil para isolar se o problema está na origem (ACR gerando o evento) ou no destino (App Service recusando a autenticação)

## Créditos

O conteúdo estático do site (`index.html` e páginas relacionadas) é material de treinamento de terceiros, do instrutor **Higor Luis Barbosa**, distribuído sob [licença Apache 2.0](./LICENSE). Este projeto o utiliza como carga de trabalho de teste para a prática de infraestrutura Azure, sem qualquer alteração ao conteúdo original.

## Autor

Michel Lepine — estudos de certificação AZ-104 (Microsoft Azure Administrator).
