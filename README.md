# DimDim CP3 — Pipeline CI/CD & CRUD Jokenpo

> **3º Checkpoint — 2º Semestre**
> API Flask (Jokenpo) integrada ao Azure SQL Server, com esteira CI/CD automatizada via Azure DevOps e deploy em nuvem via Azure Container Instances e Web App Services.

---

## Integrante

 Lucas Gomes de Araujo Lopes | RM 559607

---

## Estrutura do Repositório

```
cp3-docker-ci-cd/
├── app/
│   ├── app.py                  # Aplicação Flask principal
│   ├── models.py               # Models SQLAlchemy (Jogador, Partida)
│   └── requirements.txt        # Dependências Python
├── infra/
│   ├── script_1_acr.sh         # Script: Grupo de Recursos + ACR
│   └── script_2_aci_webapp.sh  # Script: ACI + Web App Service
├── sql/
│   └── script.sql              # DDL das tabelas no Azure SQL Server
├── postman/
│   ├── jogadores.json          # Collection CRUD de Jogadores
│   └── partidas.json           # Collection CRUD de Partidas
├── Dockerfile                  # Imagem Docker (Python 3.9 + pymssql)
├── azure-pipelines.yml         # Pipeline CI (YAML)
└── README.md
```

---

## Infraestrutura Provisionada

| Recurso | Nome | Região |
|---------|------|--------|
| Resource Group | `cp3-docker-ci-cd` | Brazil South |
| Container Registry | `pythonmssqlrm559366` | Brazil South |
| SQL Server | `sql-server-dimdim-rm559366-brazilsouth` | Brazil South |
| Banco de Dados | `db-dimdim` | — |
| Container Instance | `pythonmssqlrm559366` | Brazil South |
| App Service Plan | `planACRWebApp` (SKU F1 Linux) | Brazil South |
| Web App | `acrwebapprm559366` | Brazil South |

---

## Scripts de Automação de Infraestrutura

Toda a infraestrutura foi criada via dois scripts shell. Para executá-los, conceda permissão e rode:

```bash
chmod 700 script_1_acr.sh script_2_aci_webapp.sh
./script_1_acr.sh
./script_2_aci_webapp.sh
```

### Script 1 — Grupo de Recursos + Azure Container Registry (ACR)

Valida e provisiona o grupo base e o repositório privado de imagens Docker, habilitando o usuário administrador.

```bash
#!/bin/bash
### Variáveis
grupoRecursos=cp3-docker-ci-cd
regiao=brazilsouth
rm=rm559366
nomeACR="pythonmssql$rm"
skuACR=Basic

### Criação do Grupo de Recursos
if [ $(az group exists --name $grupoRecursos) = true ]; then
    echo "O grupo de recursos $grupoRecursos já existe"
else
    az group create --name $grupoRecursos --location $regiao
    echo "Grupo de recursos $grupoRecursos criado na localização $regiao"
fi

### Criação do Azure Container Registry
if az acr show --name $nomeACR --resource-group $grupoRecursos &> /dev/null; then
    echo "O ACR $nomeACR já existe"
else
    az acr create --resource-group $grupoRecursos --name $nomeACR --sku $skuACR
    echo "ACR $nomeACR criado com sucesso"
    az acr update --name $nomeACR --resource-group $grupoRecursos --admin-enabled true
    echo "Habilitado com sucesso o usuário Administrador para o ACR $nomeACR"
fi

ADMIN_USER=$(az acr credential show --name $nomeACR --query "username" -o tsv)
ADMIN_PASSWORD=$(az acr credential show --name $nomeACR --query "passwords[0].value" -o tsv)
export ACR_ADMIN_USER=$ADMIN_USER
export ACR_ADMIN_PASSWORD=$ADMIN_PASSWORD
```

### Script 2 — ACI + Web App Service

Cria o container no Azure Container Instances (injetando as variáveis de ambiente do banco) e provisiona o Web App com o plano F1 Linux.

```bash
#!/bin/bash
### Variáveis
grupoRecursos=cp3-docker-ci-cd
rm=rm559366
nomeACR="pythonmssql$rm"
imageACR="pythonmssql$rm.azurecr.io/pythonsql:latest"
serverACR="pythonmssql$rm.azurecr.io"
userACR=$(az acr credential show --name $nomeACR --query "username" -o tsv)
passACR=$(az acr credential show --name $nomeACR --query "passwords[0].value" -o tsv)
nomeACI="pythonmssql$rm"
regiao=brazilsouth
planService=planACRWebApp
sku=F1
appName="acrwebapp$rm"
port=80

### Criação do ACI (injetando variáveis de ambiente do banco)
az container create \
    --resource-group $grupoRecursos \
    --name $nomeACI \
    --image $imageACR \
    --cpu 2 \
    --memory 2 \
    --os-type Linux \
    --registry-login-server $serverACR \
    --registry-username $userACR \
    --registry-password $passACR \
    --dns-name-label $nomeACI \
    --restart-policy Always \
    --ports 80 \
    --environment-variables \
      DB_USER="user-dimdimsql" \
      DB_PASSWORD="Jokenpo@2026!" \
      DB_HOST="sql-server-dimdim-rm559366-brazilsouth.database.windows.net" \
      DB_NAME="db-dimdim"

### Plano de Serviço
if az appservice plan show --name $planService --resource-group $grupoRecursos &> /dev/null; then
    echo "O plano de serviço $planService já existe"
else
    az appservice plan create --name $planService --resource-group $grupoRecursos --is-linux --sku $sku
    echo "Plano de serviço $planService criado com sucesso"
fi

### Web App
if az webapp show --name $appName --resource-group $grupoRecursos &> /dev/null; then
    echo "O Serviço de Aplicativo $appName já existe"
else
    az webapp create --resource-group $grupoRecursos --plan $planService --name $appName --deployment-container-image-name $imageACR
    echo "Serviço de Aplicativo $appName criado com sucesso"
fi

### Configuração de porta
if az webapp show --name $appName --resource-group $grupoRecursos > /dev/null 2>&1; then
    az webapp config appsettings set --resource-group $grupoRecursos --name $appName --settings WEBSITES_PORT=$port
    echo "Serviço de Aplicativo $appName configurado para escutar na porta $port"
fi
```

---

## Fluxo CI/CD no Azure DevOps

O ciclo de vida completo foi gerenciado dentro do ecossistema do Azure DevOps:

1. **Azure Boards** — Task `#96: Implementar Pipeline no Azure DevOps` criada e acompanhada até a conclusão.
2. **Branch de feature** — Desenvolvimento realizado na branch `Teste_da_Pipeline`, isolando o trabalho da `main`.
3. **Pull Request** — Abertura, revisão e merge para `main`, disparando automaticamente a trigger de CI.
4. **Pipeline CI** — O arquivo `azure-pipelines.yml` executa o build e publica a imagem Docker no ACR.
5. **Release CD** — A esteira de Release dispara a task *Azure Web App for Containers*, atualizando a imagem em produção no Web App.

---

## URLs da Aplicação em Nuvem

| Ambiente | URL Base |
|----------|----------|
| Azure Container Instances | `http://pythonmssqlrm559366.brazilsouth.azurecontainer.io` |
| Azure Web App Services | `http://acrwebapprm559366.azurewebsites.net` |

> Os exemplos abaixo usam a URL do **Container Instances**. Substitua pela URL do **Web App** quando quiser testar naquele ambiente.

---

## Testes de CRUD via Postman

### CRUD de Jogadores

#### Cadastrar um jogador (Create)

- **Método:** `POST`
- **URL:** `http://pythonmssqlrm559366.brazilsouth.azurecontainer.io/jogadores`
- **Body:** selecione `raw` → tipo `JSON`

```json
{
  "nome": "Lucas"
}
```

> **Resposta esperada:** `201 Created` com o objeto do jogador criado, incluindo o `id` gerado.

---

#### Listar todos os jogadores (Read — lista completa)

- **Método:** `GET`
- **URL:** `http://pythonmssqlrm559366.brazilsouth.azurecontainer.io/jogadores`
- **Body:** nenhum (`none`)

> **Resposta esperada:** Array JSON com todos os jogadores cadastrados no banco (ex: Lucas, Gabriel…).

---

#### Buscar um jogador específico (Read — detalhe)

- **Método:** `GET`
- **URL:** `http://pythonmssqlrm559366.brazilsouth.azurecontainer.io/jogadores/4`
  *(substitua `4` pelo ID do jogador desejado)*
- **Body:** nenhum (`none`)

> **Resposta esperada:** JSON apenas com os dados do jogador solicitado.

---

#### Atualizar o nome de um jogador (Update)

- **Método:** `PUT`
- **URL:** `http://pythonmssqlrm559366.brazilsouth.azurecontainer.io/jogadores/4`
- **Body:** `raw` → `JSON`

```json
{
  "nome": "Gabriel Silva"
}
```

> **Resposta esperada:** `200 OK` com o jogador atualizado.

---

#### Deletar um jogador (Delete)

- **Método:** `DELETE`
- **URL:** `http://pythonmssqlrm559366.brazilsouth.azurecontainer.io/jogadores/4`
- **Body:** nenhum (`none`)

> **Resposta esperada:** Mensagem confirmando a exclusão.
> Devido ao `cascade="all, delete-orphan"` no model, deletar um jogador **remove automaticamente todas as partidas associadas a ele**.

---

### CRUD de Partidas (Jokenpo)

#### Registrar uma jogada (Create)

Com o `id` do jogador em mãos (ex: `1`), abra uma nova aba no Postman:

- **Método:** `POST`
- **URL:** `http://pythonmssqlrm559366.brazilsouth.azurecontainer.io/play`
- **Body:** `raw` → `JSON`

```json
{
  "jogador_id": 1,
  "choice": "pedra"
}
```

> **Resposta esperada:** O servidor computa a lógica do Jokenpo (Pedra, Papel ou Tesoura), registra a partida no banco vinculando o ID do jogador, e retorna o placar atualizado de vitórias, derrotas e empates.

---

#### Listar todas as partidas (Read)

- **Método:** `GET`
- **URL:** `http://pythonmssqlrm559366.brazilsouth.azurecontainer.io/partidas`
- **Body:** nenhum (`none`)

> **Resposta esperada:** Array com o histórico completo de todas as partidas registradas.

---

#### Filtrar partidas de um jogador específico (Read — filtrado)

- **Método:** `GET`
- **URL:** `http://pythonmssqlrm559366.brazilsouth.azurecontainer.io/partidas?jogador_id=4`
- **Body:** nenhum (`none`)

> **Resposta esperada:** Apenas as partidas do jogador com o ID informado.

---

#### Deletar uma partida específica (Delete)

- **Método:** `DELETE`
- **URL:** `http://pythonmssqlrm559366.brazilsouth.azurecontainer.io/partidas/18`
  *(substitua `18` pelo ID da partida desejada)*
- **Body:** nenhum (`none`)

> **Resposta esperada:** Mensagem confirmando a exclusão da partida.

---

#### Resetar o placar de um jogador (Delete em cascata)

- **Método:** `POST`
- **URL:** `http://pythonmssqlrm559366.brazilsouth.azurecontainer.io/jogadores/4/reset`
- **Body:** nenhum (`none`)

> **Resposta esperada:** Todas as partidas do jogador `4` são removidas, zerando o placar dele.

---

## Vídeo de Evidências

O vídeo demonstra o funcionamento fim a fim — execução do projeto e das pipelines — sem cortes, conforme exigido pelo checkpoint.

> **[Assista ao vídeo de evidências do Projeto DimDim](#)**
> *(substitua o `#` pelo link do vídeo)*

O vídeo cobre:
- Execução do projeto de ponta a ponta
- Execução/Deploy das pipelines e Release CI e CD no Azure DevOps
- Evidências do CRUD nas **duas tabelas** diretamente no banco em nuvem (Azure SQL Server)
