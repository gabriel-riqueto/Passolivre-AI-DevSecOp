# PassoLivre AI — Secure DevOps & Cloud Computing

Projeto acadêmico da disciplina Secure DevOps & Cloud Computing (FIAP).

**Repositório:** https://github.com/gabriel-riqueto/Passolivre-AI-DevSecOp  
**Branch padrão:** `main`  
**API publicada na Azure (Swagger):** https://ca-passolivre-ai.orangeflower-4427298c.brazilsouth.azurecontainerapps.io/docs

A API utiliza autenticação por `X-API-Key` nas rotas `/fontes`, `/buscar` e `/perguntar` quando publicada na Azure. O Swagger (`/docs`) e o endpoint `/health` podem ser acessados sem chave. A credencial é fornecida separadamente pela equipe e não está no repositório. A disponibilidade da demonstração depende dos recursos e dos créditos da assinatura acadêmica.

## Equipe

| Integrante | RM |
| --- | --- |
| Leticia Resina | 98069 |
| Sabrina Flores | 550781 |
| Gabriel Riqueto | 98685 |

## Escopo

A aplicação utiliza FastAPI e Ollama para responder perguntas em português com o modelo `qwen2.5:3b-instruct-q4_K_M`. Os modos de consulta são `puro` (LLM diretamente), `rag` (recuperação lexical de contexto com TF-IDF) e `agente` (recuperação, consulta a ocorrências e regras de validação/abstenção). A base demonstrativa inclui 17 trechos. As ocorrências e os endereços utilizados nos testes são fictícios.

A entrega de DevSecOps abrange containerização, infraestrutura como código, esteira de segurança e CI/CD, hospedagem na Azure, controle de acesso e monitoramento. Não há detecção de obstáculos por imagem, cadastro de usuários nem utilização de ocorrências reais nesta versão.

## Arquitetura da implantação

- **Azure Container Apps:** aplicação `ca-passolivre-ai`, no ambiente `env-passolivre-standard` (Workload Profiles / Consumption), com dois contêineres na mesma réplica: `api` (0,5 vCPU / 1 GiB) e `ollama` (2 vCPU / 4 GiB).
- **Comunicação:** o HTTPS público é encaminhado à FastAPI na porta 8000; o Ollama atende internamente em `http://127.0.0.1:11434`, sem ingress externo próprio.
- **Segredos:** o valor da API Key é referenciado como segredo do Container App. A autenticação é habilitada na Azure por `PASSOLIVRE_REQUIRE_API_KEY=true`.
- **Terraform:** gerencia os recursos de infraestrutura e o State remoto no Azure Blob Storage.
- **GitHub Actions:** executa os testes e scanners, publica a imagem da API no GHCR e atualiza o Container App via OIDC após aprovação da esteira.
- **Azure Monitor e Log Analytics:** acompanham latência, volume de requisições, categorias HTTP, CPU e memória; uma regra de alerta detecta volume elevado de respostas 4xx.

## Como executar localmente

### Pré-requisitos

Git, Docker Desktop com contêineres Linux e Docker Compose. Os comandos a seguir são para **PowerShell**. O clone padrão do repositório já utiliza a branch `main`: não é necessário escolher ou trocar de branch.

### 1. Clonar e iniciar os serviços

Execute no PowerShell, em uma pasta na qual queira guardar o projeto:

```powershell
git clone https://github.com/gabriel-riqueto/Passolivre-AI-DevSecOp.git
Set-Location .\Passolivre-AI-DevSecOp
docker compose up -d --build
```

### 2. Disponibilizar o modelo

Execute na **raiz do projeto clonado**:

```powershell
docker compose exec ollama ollama pull qwen2.5:3b-instruct-q4_K_M
docker compose exec ollama ollama list
docker compose ps
```

O download do modelo é necessário na primeira execução quando o volume de dados do Ollama está vazio. O primeiro processamento pode demorar mais durante o carregamento do modelo.

### 3. Confirmar o funcionamento da API

Execute na **raiz do projeto clonado**:

```powershell
curl.exe -i http://127.0.0.1:8000/health
```

Abra `http://127.0.0.1:8000/docs`, selecione `POST /perguntar` → **Try it out** e envie:

```json
{
  "pergunta": "A ocorrência OC002 já foi resolvida?",
  "modo": "agente"
}
```

No ambiente local padrão, a autenticação por API Key não é exigida. **Não utilize essa configuração em uma exposição pública.**

### 4. Parar a execução local

Execute na **raiz do projeto clonado**:

```powershell
docker compose down
```

Os dados do volume Docker são preservados por esse comando; não use `--volumes` se quiser manter o modelo baixado.

### Alternativa: executar com Python

Requer Python 3.11, dependências do `requirements.txt` e Ollama instalado e iniciado no computador. Execute na **raiz do projeto clonado**:

```powershell
py -3.11 -m venv .venv
.\.venv\Scripts\python.exe -m pip install -r requirements.txt
.\.venv\Scripts\python.exe -m pytest -q
ollama pull qwen2.5:3b-instruct-q4_K_M
.\.venv\Scripts\python.exe -m uvicorn app.main:app --host 127.0.0.1 --port 8000
```

Se o serviço Ollama não estiver iniciado, execute `ollama serve` em outro terminal. Não execute Compose e Uvicorn simultaneamente na mesma porta 8000.

## Rotas da API

| Rota | Método | Finalidade |
| --- | --- | --- |
| `/health` | GET | Verificar a disponibilidade da FastAPI (não valida o Ollama). |
| `/docs` | GET | Documentação Swagger. |
| `/fontes` | GET | Listar fontes do corpus. |
| `/buscar` | GET | Pesquisar contexto na base. |
| `/perguntar` | POST | Enviar uma pergunta ao sistema. |

No ambiente Azure, `/fontes`, `/buscar` e `/perguntar` exigem o cabeçalho `X-API-Key`. A chave **não deve ser incluída em commits, logs, capturas ou documentos públicos**. Para testar as rotas protegidas, o professor poderá inserir a chave recebida por canal privado no botão **Authorize** do Swagger.

Para conferir a disponibilidade pública sem credencial, execute em qualquer PowerShell:

```powershell
curl.exe -s -o NUL -w "Health HTTP %{http_code}\n" https://ca-passolivre-ai.orangeflower-4427298c.brazilsouth.azurecontainerapps.io/health
curl.exe -s -o NUL -w "Sem chave HTTP %{http_code}\n" https://ca-passolivre-ai.orangeflower-4427298c.brazilsouth.azurecontainerapps.io/fontes
```

Nos testes registrados, os resultados foram `200` e `401`, respectivamente.

## Pipeline DevSecOps e deploy automatizado

**Arquivo do workflow:** `.github/workflows/devsecops.yml`, na branch padrão `main`.

O pipeline é acionado em alterações de `main` e realiza:

1. Instalação de dependências e testes com `pytest`.
2. Detecção de segredos com Gitleaks e análise estática com Semgrep.
3. Construção da imagem Docker e varredura de vulnerabilidades com Trivy, incluindo relatório completo.
4. Bloqueio de vulnerabilidades HIGH/CRITICAL **com correção disponível**; publicação da imagem no GHCR com tags `latest` e SHA do commit quando as verificações passam.
5. Autenticação na Azure por OIDC e atualização do contêiner `api` de `ca-passolivre-ai` depois da aprovação do job de segurança.

A política do Trivy mantém uma exceção para problemas ainda sem correção (`ignore-unfixed: true`). Portanto, uma execução aprovada não significa ausência absoluta de vulnerabilidades. O relatório completo fica disponível como evidência do scan.

**Configuração necessária para quem for reproduzir o CI/CD em outra assinatura:** cadastrar `AZURE_CLIENT_ID`, `AZURE_TENANT_ID` e `AZURE_SUBSCRIPTION_ID` em GitHub Actions Secrets; configurar uma credencial federada compatível com o repositório e a branch `main`; e conceder as permissões Azure e GHCR necessárias. Esses identificadores não são a API Key da aplicação. O deploy não estará autorizado numa conta externa somente por clonar o código.

**Imagem da API:** `ghcr.io/gabriel-riqueto/passolivre-ai`  
**Imagem de Ollama usada na implantação:** `ghcr.io/gabriel-riqueto/passolivre-ollama-cpu`

## Infraestrutura com Terraform

| Arquivo | Responsabilidade |
| --- | --- |
| `infra/providers.tf` | Provedores AzureRM, AzAPI e Random; backend remoto Azure. |
| `infra/main.tf` | Recursos básicos, incluindo Resource Group e Log Analytics. |
| `infra/environment-standard.tf` | Ambiente de Container Apps no modo Workload Profiles. |
| `infra/container-app.tf` | Contêineres, ingress, recursos de CPU/memória, variáveis e segredos. |
| `infra/api-key.tf` | Geração da chave de acesso usada na implantação Azure. |
| `infra/outputs.tf` | Saídas de infraestrutura. |

Para **validar a configuração sem acessar o backend remoto** (necessário Terraform instalado), execute na raiz do projeto:

```powershell
terraform -chdir=infra init -backend=false
terraform -chdir=infra fmt -check
terraform -chdir=infra validate
```

**Não execute `terraform apply` ao clonar o projeto.** O backend remoto e a criação/atualização de recursos exigem uma assinatura Azure, RBAC e parâmetros específicos. O State pode conter dados sensíveis e não deve ser publicado. A infraestrutura já implantada utiliza `rg-passolivre-devops`, `env-passolivre-standard`, `ca-passolivre-ai` e `logs-passolivre` na região Brazil South.

## Monitoramento

No Azure Monitor, o dashboard **Passo Livre AI — Observabilidade DevSecOps** apresenta tempo médio de resposta, tráfego, requisições por categoria HTTP e utilização de CPU e memória. A regra `alert-passolivre-acessos-negados` acompanha respostas HTTP `4xx`: ela foi configurada para disparar quando houver mais de cinco requisições em uma janela de cinco minutos e foi validada com acessos de teste que retornaram `401`. Erros 4xx também podem resultar de comportamentos esperados e não representam, isoladamente, ataques.

## Privacidade e LGPD

A demonstração utiliza ocorrências fictícias e não exige cadastro de usuários. Como os campos de pergunta aceitam texto livre, **não encaminhe dados pessoais reais à instância de demonstração**. O processamento da IA ocorre nos contêineres FastAPI e Ollama da implantação em Brazil South, mas serviços externos e registros administrativos podem ter políticas e localidades distintas. 

O controle por API Key reduz acessos não autorizados, mas não substitui políticas de retenção, governança, controles adicionais e avaliação de riscos para uso produtivo. Evite expor credenciais, prompts sensíveis e informações identificáveis em logs ou evidências.

## Custos e documentação

Durante a consulta ao Azure Cost Management, em novembro de 2026, foi identificado um custo acumulado inferior a R$ 0,01 na assinatura.

O consumo registrado estava associado principalmente ao serviço de armazenamento (Storage), utilizado pelo Terraform State. Os serviços de monitoramento e o grupo de recursos da aplicação não apresentavam custos cobrados na consulta.

A documentação técnica da disciplina, contendo diagrama da arquitetura, responsabilidade compartilhada, LGPD, custos e capturas de evidência, será disponibilizada na pasta `docs/`. P

## Referências

- [Responsabilidade compartilhada — Microsoft Learn](https://learn.microsoft.com/en-us/azure/security/fundamentals/shared-responsibility)
- [Azure Container Apps — cobrança](https://learn.microsoft.com/en-us/azure/container-apps/billing)
- [Azure Monitor — custos](https://learn.microsoft.com/en-us/azure/azure-monitor/fundamentals/cost-usage)
- [Lei Geral de Proteção de Dados — Lei 13.709/2018](https://www.planalto.gov.br/ccivil_03/_ato2015-2018/2018/lei/l13709.htm)
