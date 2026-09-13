# Mini Curso DevOps - Join API

**Do repositório vazio ao deploy em produção.**

Este é o material completo do curso. Siga na ordem: cada parte depende da anterior. Ao final
você terá uma API .NET rodando em dois ambientes na nuvem, com deploy automático a cada merge.

> **Como ler:** todo bloco de código traz um cabeçalho dizendo **qual arquivo** e **o que fazer**
> (`CRIAR`, `EDITAR`, `EXCLUIR`). Se não tem cabeçalho de arquivo, é comando de terminal.

## Onde rodar cada comando

Este curso usa **três terminais diferentes**, e eles não são intercambiáveis. Por isso, todo
bloco de comando deste material vem com um cabeçalho dizendo **em qual terminal executar**:

| Cabeçalho | O que é | Como abrir |
|---|---|---|
| **PowerShell (Windows)** | O terminal do próprio Windows | Menu Iniciar → digite `PowerShell` |
| **Terminal do VS Code** | O mesmo PowerShell, embutido no editor e já dentro da pasta do projeto | VS Code → menu **Terminal → New Terminal** (a partir da Parte 4) |
| **Terminal WSL (Ubuntu)** | O Linux que roda dentro do Windows | Menu Iniciar → `Ubuntu` (só na Parte 1); da Parte 5 em diante, **pelo VS Code** |

**A regra que resolve 90% dos casos:**

| Comando | Terminal |
|---|---|
| `git`, `dotnet`, `wsl` | Windows - PowerShell, avulso ou dentro do VS Code |
| `docker` | WSL (Ubuntu) |

> **Por que essa divisão:** o Git e o SDK do .NET você instala **no Windows**, são programas
> Windows. Já os containers rodam **dentro do WSL**: o Docker Desktop é só a interface, o motor
> que constrói e executa as imagens vive no Linux. Rodar `docker` pelo PowerShell até funciona
> em algumas instalações, mas neste curso padronizamos no WSL para que todo mundo veja
> exatamente a mesma saída.

---

## Índice

| Parte | Conteúdo | Entregável |
|---|---|---|
| [0](#parte-0---o-que-vamos-construir) | O que vamos construir | Entendimento |
| [1](#parte-1---preparando-o-ambiente) | Preparando o ambiente | Máquina pronta |
| [2](#parte-2---repositório-e-board) | Repositório e board | GitHub + Trello |
| [3](#parte-3---branches-e-gitflow) | Branches e GitFlow | `develop` + `feature/1` |
| [4](#parte-4---criando-o-projeto-net) | Criando o projeto .NET | API respondendo local |
| [5](#parte-5---containerizando-com-docker) | Containerizando com Docker | Imagem rodando |
| [6](#parte-6---primeiro-deploy-no-render) | Primeiro deploy no Render | 2 serviços criados |
| [7](#parte-7---o-deploy-quebrou-hotfix) | O deploy quebrou: hotfix | 2 ambientes no ar |
| [8](#parte-8---feature-listar-filmes) | Feature: listar filmes | `GET /filmes` |
| [9](#parte-9---feature-obter-filme-por-id) | Feature: obter por ID | `GET /filmes/{id}` |
| [10](#parte-10---encerramento) | Encerramento | Fluxo completo |

---

# PARTE 0 - O que vamos construir

## 0.1 O produto

Uma API REST de catálogo de filmes, com dois endpoints:

| Método | Rota | Retorna |
|---|---|---|
| `GET` | `/filmes` | Lista completa de filmes |
| `GET` | `/filmes/{id}` | Um filme específico, ou `404` |

A API **não tem banco de dados**. Ela busca os dados de um arquivo JSON hospedado no GitHub.
Isso é proposital: mantém o foco do curso no DevOps, não em persistência.

## 0.2 O que você vai praticar

Este curso não é sobre escrever C#. É sobre **o caminho que o código percorre** até chegar no
usuário. Você vai praticar:

- **Versionamento com GitFlow** - branches com papéis definidos (`main`, `develop`, `feature`, `hotfix`)
- **Gestão de tarefas** - cards no Trello amarrados às branches
- **Containerização** - empacotar a aplicação com Docker para que rode igual em qualquer lugar
- **CI/CD** - deploy automático disparado por merge
- **Ambientes separados** - homologação e produção com dados diferentes
- **Resposta a incidente** - um deploy vai quebrar, e você vai corrigir via `hotfix`

## 0.3 A arquitetura

A solução tem três projetos .NET, com dependências em uma direção só:

```
┌─────────┐      ┌───────────┐      ┌──────────┐
│   Api   │ ───► │  Service  │ ───► │  Domain  │
└─────────┘      └───────────┘      └──────────┘
 Controllers      Regra de           DTOs
 Swagger          negócio            (sem dependências)
 Program.cs       RestSharp
```

| Projeto | Responsabilidade | Depende de |
|---|---|---|
| **Api** | Recebe requisições HTTP, devolve respostas. Não sabe *como* os dados são obtidos. | Service, Domain |
| **Service** | Sabe buscar os filmes na fonte externa. Não sabe que existe HTTP. | Domain |
| **Domain** | Só as classes que representam os dados. Não depende de nada. | - |

**Por que separar assim?** Porque a direção da dependência protege o que é estável. O `Domain`
não muda quando você troca de fonte de dados. O `Service` não muda quando você troca de
framework web. Se tudo estivesse em um projeto só, qualquer mudança contaminaria tudo.

## 0.4 Os dois ambientes

O mesmo código roda em dois lugares, apontando para JSONs diferentes, simulando dois bancos.

| Ambiente | Serviço no Render | Branch | Fonte de dados |
|---|---|---|---|
| **Homologação (HML)** | `hml-join-api` | `develop` | `db-hml.json` |
| **Produção (PROD)** | `join-api` | `main` | `db.json` |

```
   merge em develop  ──►  hml-join-api  ──►  db-hml.json     (teste)
   merge em main     ──►  join-api      ──►  db.json         (real)
```

Toda alteração passa por HML antes de chegar em PROD. Essa é a regra do curso e a razão de
existir do GitFlow.

## 0.5 O caminho completo

```
Card no Trello
     │
     ▼
Branch feature/{id}-descricao  (a partir de develop)
     │
     ▼
Código + teste local com Docker
     │
     ▼
Push  ──►  PR para develop  ──►  DEPLOY AUTOMÁTICO EM HML
     │
     ▼
Valida em HML
     │
     ▼
PR develop → main  ──►  DEPLOY AUTOMÁTICO EM PROD
     │
     ▼
Card para "Concluído"
```

---

# PARTE 1 - Preparando o ambiente

## 1.1 WSL (só Windows, e só se você ainda não tem)

O Docker no Windows roda em cima do WSL (Windows Subsystem for Linux). Se você já tem, pule.

1. Abra o **PowerShell como Administrador** (botão direito no menu Iniciar → *Terminal (Admin)*)
2. Execute:

> **PowerShell (Windows) - como Administrador**

```powershell
wsl --install
```

3. **Reinicie o computador** quando terminar.
4. Após reiniciar, uma janela preta abre sozinha pedindo um **nome de usuário** e uma **senha**
   do Linux. Crie e **anote**, a senha não aparece enquanto você digita, é normal.

**Validar:**

> **PowerShell (Windows)**

```powershell
wsl --status
```

> **Repare:** `wsl` é um comando **do Windows**, não do Linux. É ele quem instala e administra o
> Ubuntu, por isso roda no PowerShell, e não dentro do próprio Ubuntu.

## 1.2 Docker Desktop

Se você já tem Docker Desktop instalado, pule.

Baixe e instale: <https://www.docker.com/products/docker-desktop/>

Após instalar, **abra o Docker Desktop e deixe rodando**. O Docker só funciona com o Desktop
aberto, o ícone da baleia precisa estar na bandeja do sistema.

## 1.3 Conectar o Docker ao WSL

1. Abra o Docker Desktop
2. **Settings** (engrenagem) → **Resources** → **WSL Integration**
3. Ative o toggle da sua distribuição Linux
4. **Apply & Restart**

**Validar** - agora você precisa abrir o terminal do Linux. Menu Iniciar → digite **`Ubuntu`** →
`Enter`. Abre um terminal com o prompt parecido com `seu-usuario@sua-maquina:~$`.

> **Terminal WSL (Ubuntu)**

```bash
docker --version
```

Deve aparecer algo como `Docker version 29.x.x`. Se der `command not found`, a integração não
foi aplicada, volte ao passo 3, confirme que o toggle da distribuição está ligado e feche e
abra o terminal do Ubuntu.

> A partir da Parte 5 você não vai mais abrir o Ubuntu pelo menu Iniciar, vai abri-lo **de
> dentro do VS Code**, o que resolve um problema de pasta que explico lá. Por enquanto, o menu
> Iniciar serve para esta validação.

## 1.4 .NET SDK 10

Baixe em <https://dotnet.microsoft.com/download/dotnet/10.0> e instale o **SDK** (não o Runtime).
Escolha a versão para **Windows x64**, o SDK é instalado no Windows, não dentro do WSL.

**Validar** - feche e abra o PowerShell antes (só assim ele enxerga o `PATH` novo):

> **PowerShell (Windows)**

```powershell
dotnet --version
```

Deve retornar `10.x.x`.

> **Não tente rodar `dotnet` no terminal do Ubuntu.** Ele vai responder `command not found`,
> e está certo: o SDK foi instalado no Windows. Todo comando `dotnet` deste curso roda no
> PowerShell.

## 1.5 Git

O Git é o que versiona o código. Sem ele, nada do resto do curso acontece. Instale **no
Windows**, é de lá que todos os comandos `git` deste material vão sair.

### Passo 1 - Instalar

Baixe em <https://git-scm.com/download/win>. O download começa sozinho.

Execute o instalador e **avance com o padrão** (`Next` até o fim). Duas telas merecem atenção,
as duas já vêm marcadas corretamente, só confirme:

| Tela | O que deixar marcado |
|---|---|
| **Adjusting your PATH environment** | `Git from the command line and also from 3rd-party software` (a opção do meio) |
| **Configuring the line ending conversions** | `Checkout Windows-style, commit Unix-style line endings` |

A primeira é o que faz o comando `git` funcionar no PowerShell e dentro do VS Code. A segunda
cuida da diferença de quebra de linha entre Windows e Linux, importante porque o seu código vai
rodar dentro de um container Linux.

Ao final, **reinicie o computador** (ou pelo menos feche todos os terminais abertos).

### Passo 2 - Validar a instalação

> **PowerShell (Windows)**

```powershell
git --version
```

Deve retornar algo como `git version 2.51.0.windows.1`.

Se aparecer `O termo 'git' não é reconhecido...`, o `PATH` não foi configurado: reinstale
marcando a opção da tabela acima.

### Passo 3 - Configurar sua identidade

**Todo commit carrega um nome e um e-mail.** Não é decoração: é assim que o histórico do
projeto registra quem fez o quê. Sem isso configurado, o Git **recusa o seu primeiro commit**
com um erro pedindo exatamente estes dois valores.

A configuração é **global**, vale para todos os repositórios da máquina, e você faz **uma vez
só na vida**.

> **PowerShell (Windows)**

```powershell
git config --global user.name "Seu Nome Completo"
git config --global user.email "{seu-email-institucional}"
```

| Campo | O que colocar |
|---|---|
| `user.name` | Seu **nome real**, completo. Não use apelido - é o que aparece em cada commit no GitHub |
| `user.email` | O **e-mail institucional** que você usa na faculdade - o mesmo com que vai criar a conta do GitHub na seção 1.7 |

> **Neste minicurso, use o e-mail institucional.** É ele que amarra os seus commits à sua
> conta do GitHub e ao seu registro no curso. Se você commitar com um e-mail que **não** está
> cadastrado na sua conta do GitHub, os commits aparecem como de um autor desconhecido: sem
> foto, sem link para o seu perfil e sem contar na sua atividade. Do ponto de vista do GitHub,
> não foi você quem escreveu aquele código.

Exemplo preenchido:

```powershell
git config --global user.name "Maria Silva"
git config --global user.email "maria.silva@aluno.unifenas.br"
```

### Passo 4 - Duas configurações que evitam dor de cabeça

> **PowerShell (Windows)**

```powershell
git config --global init.defaultBranch main
git config --global core.autocrlf true
```

| Configuração | O que faz |
|---|---|
| `init.defaultBranch main` | Faz todo repositório novo começar na branch `main`. O padrão antigo era `master`, e o GitHub usa `main` - alinhar os dois evita confusão |
| `core.autocrlf true` | Converte as quebras de linha: `CRLF` na sua máquina Windows, `LF` dentro do repositório. Sem isso, arquivos enviados daqui chegam no Linux com um caractere invisível a mais no fim de cada linha |

### Passo 5 - Conferir

> **PowerShell (Windows)**

```powershell
git config --global --list
```

Devem aparecer as quatro linhas que você acabou de configurar:

```text
user.name=Maria Silva
user.email=maria.silva@aluno.unifenas.br
init.defaultbranch=main
core.autocrlf=true
```

> **Errou algum valor?** Rode o mesmo `git config --global ...` de novo com o valor certo. Ele
> sobrescreve, não duplica.

### Passo 6 - Autenticação no GitHub

Os repositórios deste curso são **privados**. Na primeira vez que você clonar ou der `push`, o
**Git Credential Manager** (instalado junto com o Git) abre uma janela:

1. Clique em **Sign in with your browser**
2. O navegador abre no GitHub → faça login → **Authorize**
3. Volte ao terminal: o comando continua sozinho

Isso acontece **uma vez**. Depois a credencial fica salva no Gerenciador de Credenciais do
Windows e o Git não pergunta mais.

> **O GitHub não aceita mais a senha da sua conta no terminal.** Se alguma tela pedir
> "password", não é a senha do site, seria um token de acesso. Use sempre o fluxo pelo
> navegador descrito acima.

## 1.6 Visual Studio Code

É o editor que vamos usar. A partir da **Parte 4**, todos os comandos rodam pelo terminal
integrado dele, é o que mantém você sempre dentro da pasta certa do projeto, sem ficar
navegando com `cd`.

1. Baixe e instale: <https://code.visualstudio.com/>
2. Durante a instalação, deixe marcada a opção **"Adicionar ao PATH"** (vem marcada por padrão)
3. Abra o VS Code e instale duas extensões, ícone de blocos na barra lateral esquerda, ou
   `Ctrl` + `Shift` + `X`:

| Extensão | Para quê |
|---|---|
| **C#** (Microsoft) | Realce de sintaxe, IntelliSense e navegação no código C# |
| **WSL** (Microsoft) | Garante que o Ubuntu apareça na lista de terminais do editor - é o que usaremos na Parte 5 para rodar o Docker |

> Você ainda **não precisa abrir nenhuma pasta** no VS Code. Isso acontece na Parte 4, depois
> que o projeto existir.

## 1.7 Contas necessárias

| Serviço | Para quê | Link |
|---|---|---|
| **GitHub** | Hospedar o código | <https://github.com> |
| **Trello** | Board de tarefas | <https://trello.com> |
| **Render** | Hospedar a API | <https://dashboard.render.com> |

Crie as três agora.

- No **GitHub**, cadastre-se com o **mesmo e-mail institucional** que você colocou no
  `git config` da seção 1.5. É o que faz seus commits aparecerem vinculados ao seu perfil.
- No **Render**, cadastre-se **com a conta do GitHub**, isso facilita a conexão depois.

### Checkpoint 1

- [ ] `wsl --status` responde no **PowerShell**
- [ ] `docker --version` responde no **terminal do Ubuntu**
- [ ] `dotnet --version` retorna 10.x no **PowerShell**
- [ ] `git --version` responde no **PowerShell**
- [ ] `git config --global --list` mostra seu nome e o **e-mail institucional**
- [ ] Docker Desktop aberto e rodando
- [ ] VS Code instalado, com as extensões **C#** e **WSL**
- [ ] Contas GitHub, Trello e Render criadas (GitHub com o e-mail institucional)

---

# PARTE 2 - Repositório e board

## 2.1 Criar o repositório da API

No GitHub, clique em **New repository** e preencha:

| Campo | Valor |
|---|---|
| **Repository name** | `join-api` |
| **Visibility** | **Private** |
| **Add a README file** | Sim, marcado |
| **Add .gitignore** | `VisualStudio` (template do .NET) |

Clique em **Create repository**.

> **Por que o `.gitignore` importa:** ele impede que arquivos gerados pela compilação
> (`bin/`, `obj/`) e segredos (`.env`) sejam enviados para o GitHub. Sem ele, seu repositório
> encheria de lixo binário e você poderia vazar credenciais.

## 2.2 Clonar na sua máquina

Crie uma pasta para o curso e clone o repositório **dentro dela**.

> **PowerShell (Windows)**

```powershell
cd $HOME\Documents
mkdir Join
cd Join

git clone https://github.com/{seu-usuario}/join-api.git
```

> **No `git clone`, a janela do Git Credential Manager vai abrir**, é a autenticação
> da seção 1.5, passo 6. Clique em **Sign in with your browser**, autorize no GitHub e volte ao
> terminal. O clone continua sozinho, e nos próximos comandos ele não pergunta mais.

Estrutura resultante:

```
Join/
└── join-api/     ← vamos trabalhar aqui
```

> **Daqui em diante, todo comando é executado dentro de `join-api/`**, a menos que o texto
> diga o contrário. Entre nela agora: `cd join-api`

## 2.3 Criar o board no Trello

1. Acesse <https://trello.com> e vá em **Quadros**
2. Procure o template **"Kanban Quadro Modelo"**
3. Clique em **Criar quadro com base em template**
4. Preencha:
   - **Título:** `Join`
   - **Área de trabalho:** selecione a sua
   - Deixe **apenas** `Manter cartões de template` marcado
5. **Criar**
6. Entre no quadro
7. **Apague todas as colunas exceto três**, deixando: `Backlog`, `Em andamento`, `Concluído`
8. Clique em **Power-Ups** → busque **"Board and Card Keys"** → **Adicionar**

> **Para que serve o Power-Up:** ele dá um **número** a cada card (`#1`, `#2`, `#3`…). Vamos usar
> esse número no nome das branches. É assim que se amarra código a tarefa: olhando a branch
> `feature/4-criacao-api-listar-filmes`, qualquer pessoa sabe que ela resolve o card #4.

## 2.4 Criar os cards

Crie estes cards na coluna **Backlog**:

| Card | Título |
|---|---|
| 1 | Projeto base |
| 2 | Configuração de deploy |
| 3 | Criação da API de listar filmes |
| 4 | Criação da API de obter filme por ID |

Anote os números que o Trello atribuiu, você vai usá-los nos nomes das branches. Os números
do seu board podem ser diferentes dos deste material; **use sempre os seus**.

Neste material, vou me referir a eles como `{id-card}`.

### Checkpoint 2

- [ ] `join-api` criado e privado no GitHub
- [ ] Clonado dentro da pasta `Join/`
- [ ] Board `Join` no Trello com 3 colunas
- [ ] Power-Up "Board and Card Keys" ativo
- [ ] 4 cards criados no Backlog

---

# PARTE 3 - Branches e GitFlow

## 3.1 Entendendo o GitFlow

GitFlow é um acordo sobre **o que cada branch significa**. Não é uma ferramenta, é uma
convenção que o time combina e respeita.

| Branch | Significado | Nasce de | Vai para |
|---|---|---|---|
| `main` | O que está em **produção**, agora | - | - |
| `develop` | O que está pronto para **testar** | `main` | `main` |
| `feature/*` | Uma funcionalidade em construção | `develop` | `develop` |
| `hotfix/*` | Correção **urgente** de produção | `main` | `main` **e** `develop` |

```
main      ●───────────────●───────────────────●
          │               ▲                   ▲
          │               │ hotfix            │
develop   ●───────●───────●─────────●─────────●
                  ▲                 ▲
                  │ feature         │ feature
```

**A regra que mais se esquece:** um `hotfix` precisa de **dois merges**, em `main` (para
consertar produção) e em `develop` (para o bug não voltar no próximo release). Vamos praticar
isso na Parte 7.

## 3.2 Criar a branch develop

Dentro de `join-api/`, se você ainda não entrou, rode `cd join-api` primeiro:

> **PowerShell (Windows)**

```powershell
git checkout -b develop main
git push -u origin develop
```

**O que aconteceu:**
- `checkout -b develop main` - cria a branch `develop` a partir de `main` e já muda para ela
- `push -u origin develop` - envia para o GitHub e **vincula** a branch local à remota. O `-u`
  faz com que futuros `git push` funcionem sem argumentos

## 3.3 Criar a branch da primeira feature

> **PowerShell (Windows)**

```powershell
git checkout -b feature/{id-card}-projeto-base develop
git push -u origin feature/{id-card}-projeto-base
```

Substitua `{id-card}` pelo número do card **"Projeto base"** no seu Trello.

Mova o card para **Em andamento**.

**Validar onde você está:**

> **PowerShell (Windows)**

```powershell
git branch
```

O `*` deve estar em `feature/{id-card}-projeto-base`.

### Checkpoint 3

- [ ] `develop` existe local e no GitHub
- [ ] `feature/{id}-projeto-base` existe local e no GitHub
- [ ] Você está **na branch da feature**
- [ ] Card "Projeto base" em *Em andamento*

---

# PARTE 4 - Criando o projeto .NET

## 4.0 Abrir o projeto no VS Code

**Deste ponto em diante, pare de usar o PowerShell avulso.** A partir de agora você vai criar e
editar arquivos, e o terminal precisa estar **sempre** na raiz do projeto. O terminal integrado
do VS Code resolve isso de graça: ele já abre dentro da pasta que você abriu no editor.

1. Abra o **Visual Studio Code**
2. Menu **File → Open Folder...**
3. Selecione a pasta **`join-api`**, a pasta do repositório, **não** a `Join/` de fora
4. Se aparecer *"Do you trust the authors of the files in this folder?"*, clique em
   **Yes, I trust the authors**
5. Abra o terminal: menu **Terminal → New Terminal** (atalho `Ctrl` + `` ` ``)

O terminal abre na parte de baixo da tela. **Confira o prompt:** ele deve terminar em
`\join-api>`. Esse é o sinal de que você está no lugar certo.

> **Por que isso importa:** todos os comandos das próximas partes assumem que você está na raiz
> de `join-api/`. Rodar `dotnet new sln` na pasta errada cria a solution no lugar errado, e o
> erro só aparece bem depois, quando algum caminho não bate. Abrindo o terminal pelo VS Code,
> a pasta certa já é a pasta corrente e você nunca mais precisa se preocupar com `cd`.

**De agora em diante, o cabeçalho "Terminal do VS Code" significa exatamente este terminal**,
que é um PowerShell rodando dentro do editor. Tudo o que você aprendeu a rodar no PowerShell
continua funcionando igual aqui.

## 4.1 Criar a solution e os projetos

Ainda em `join-api/`, execute na ordem:

> **Terminal do VS Code (PowerShell)**

```powershell
dotnet new sln
dotnet new webapi -n Api --use-controllers
dotnet new classlib -n Service
dotnet new classlib -n Domain
```

**O que cada comando faz:**

| Comando | Cria |
|---|---|
| `dotnet new sln` | O arquivo de solution (`join-api.slnx`) - um índice que agrupa os projetos |
| `dotnet new webapi -n Api --use-controllers` | O projeto web. `--use-controllers` gera a estrutura com Controllers, em vez de Minimal APIs |
| `dotnet new classlib -n Service` | Biblioteca de classes - compila para `.dll`, não roda sozinha |
| `dotnet new classlib -n Domain` | Idem |

## 4.2 Registrar os projetos na solution

> **Terminal do VS Code (PowerShell)**

```powershell
dotnet sln add ./Api/Api.csproj
dotnet sln add ./Service/Service.csproj
dotnet sln add ./Domain/Domain.csproj
```

Sem isso, a solution existe mas está vazia, abrir no Visual Studio não mostraria os projetos.

## 4.3 Ligar os projetos entre si

> **Terminal do VS Code (PowerShell)**

```powershell
dotnet add ./Api/Api.csproj reference ./Service/Service.csproj
dotnet add ./Api/Api.csproj reference ./Domain/Domain.csproj
dotnet add ./Service/Service.csproj reference ./Domain/Domain.csproj
```

Isso implementa a direção de dependência da seção 0.3:

```
Api ──► Service ──► Domain
Api ──────────────► Domain
```

Note o que **não** existe: nenhuma referência de `Domain` ou `Service` para `Api`. Se você
tentasse criar, o .NET recusaria, seria uma referência circular.

## 4.4 Excluir os arquivos do template

O template gera arquivos de exemplo que não vamos usar.

> **EXCLUIR** - três arquivos:
> - `Service/Class1.cs`
> - `Domain/Class1.cs`
> - `Api/WeatherForecast.cs`

Apague os três de uma vez pelo terminal:

> **Terminal do VS Code (PowerShell)**

```powershell
Remove-Item Service/Class1.cs, Domain/Class1.cs, Api/WeatherForecast.cs
```

> **Repare na vírgula.** No PowerShell, vários arquivos vão separados por **vírgula**, diferente
> do `rm` do Linux, que separa por espaço. Se preferir, apague os três pelo painel de arquivos do
> VS Code, com `Delete`. Dá no mesmo.

## 4.5 Renomear o controller de exemplo

> **RENOMEAR** - `Api/Controllers/WeatherForecastController.cs` → `Api/Controllers/FilmesController.cs`

Pelo terminal:

> **Terminal do VS Code (PowerShell)**

```powershell
Rename-Item Api/Controllers/WeatherForecastController.cs FilmesController.cs
```

> No `Rename-Item` o segundo argumento é **só o novo nome**, sem o caminho, o arquivo continua
> na mesma pasta. Se preferir, botão direito no arquivo, no painel lateral do VS Code, →
> **Rename**.

## 4.6 Escrever o primeiro controller

> **EDITAR** - `Api/Controllers/FilmesController.cs`
> Apague **todo** o conteúdo e substitua por:

```csharp
using Microsoft.AspNetCore.Mvc;

namespace Api.Controllers;

[ApiController]
[Route("[controller]")]
public class FilmesController : ControllerBase
{
    [HttpGet]
    public IActionResult Get() => Ok("Deu certo!");
}
```

**Linha a linha:**

| Trecho | O que faz |
|---|---|
| `[ApiController]` | Marca a classe como controller de API. Ativa validação automática de modelo e respostas de erro padronizadas |
| `[Route("[controller]")]` | Define a URL. `[controller]` é substituído pelo nome da classe **sem o sufixo "Controller"** → `FilmesController` vira a rota `/filmes` |
| `ControllerBase` | Classe base para APIs. (`Controller`, sem o `Base`, é para MVC com views - não precisamos) |
| `[HttpGet]` | Este método responde a requisições `GET` |
| `IActionResult` | Permite retornar qualquer código HTTP. `Ok(...)` devolve `200` |

Por enquanto retorna um texto fixo. É só para provar que a API sobe.

## 4.7 Instalar o Swagger

> **Terminal do VS Code (PowerShell)**

```powershell
dotnet add ./Api/Api.csproj package Swashbuckle.AspNetCore
```

**Por que:** o template do .NET 10 vem com `Microsoft.AspNetCore.OpenApi`, que **gera** o
documento OpenAPI mas **não tem interface visual**. O Swashbuckle adiciona o Swagger UI, a
página onde você testa os endpoints pelo navegador. Para um curso, ver e clicar vale mais do
que um JSON.

## 4.8 Configurar o Program.cs

O `Program.cs` é o ponto de entrada: ele monta a aplicação e define o pipeline de requisições.

> **EDITAR** - `Api/Program.cs`
> Substitua **todo** o conteúdo por:

```csharp
var builder = WebApplication.CreateBuilder(args);

builder.Services.AddControllers();
builder.Services.AddEndpointsApiExplorer();
builder.Services.AddSwaggerGen();

var app = builder.Build();

app.UseSwagger();
app.UseSwaggerUI();

app.UseHttpsRedirection();

app.UseAuthorization();

app.MapControllers();

app.Run();
```

**O que mudou em relação ao template:**

| Antes (template) | Depois | Por quê |
|---|---|---|
| `builder.Services.AddOpenApi();` | `AddEndpointsApiExplorer();`<br>`AddSwaggerGen();` | Troca o gerador nativo pelo Swashbuckle |
| `if (app.Environment.IsDevelopment()) { app.MapOpenApi(); }` | `app.UseSwagger();`<br>`app.UseSwaggerUI();` | Remove a condição - queremos o Swagger **também em produção**, para validar o deploy pelo navegador |

**Entendendo as duas metades do arquivo:**

```csharp
var builder = WebApplication.CreateBuilder(args);
// ─── REGISTRO ───
// Tudo que é builder.Services.Add... acontece ANTES do Build().
// Aqui você diz "estes serviços existem".

var app = builder.Build();
// ─── PIPELINE ───
// Tudo que é app.Use... acontece DEPOIS.
// Aqui você define a ORDEM pela qual cada requisição passa.
```

A ordem do pipeline importa: `app.MapControllers()` precisa vir **por último**, porque é ele
quem finalmente entrega a requisição ao seu controller.

> `app.UseHttpsRedirection()` está aqui de propósito. Vamos removê-lo na Parte 7, guarde
> essa informação.

## 4.9 Rodar e validar

> **Terminal do VS Code (PowerShell)**

```powershell
cd Api
dotnet run
```

O terminal mostra a porta, algo como `http://localhost:5142`.

Abra no navegador: **`http://localhost:5142/swagger`**

Você deve ver a interface do Swagger com o endpoint `GET /Filmes`. Clique em
**Try it out** → **Execute**. A resposta deve ser `"Deu certo!"` com status `200`.

Pare com `Ctrl+C` e volte para a raiz:

> **Terminal do VS Code (PowerShell)**

```powershell
cd ..
```

### Checkpoint 4

- [ ] `dotnet build` compila sem erros
- [ ] `/swagger` abre no navegador
- [ ] `GET /Filmes` retorna `"Deu certo!"`

---

# PARTE 5 - Containerizando com Docker

## 5.1 Por que Docker

Sua máquina tem o SDK do .NET, suas variáveis, seu sistema operacional. O servidor não tem nada
disso. **Container** é a resposta: um pacote que carrega a aplicação *e* tudo que ela precisa
para rodar. O mesmo pacote roda igual na sua máquina, na do colega e no servidor.

Dois conceitos:

- **Imagem** - o pacote pronto, imutável. Como um instalador.
- **Container** - uma imagem em execução. Como o programa aberto.

O **Dockerfile** é a receita que constrói a imagem.

## 5.2 Criar o Dockerfile

> **CRIAR** - `Dockerfile` (na raiz de `join-api/`, ao lado do `.slnx`, **sem extensão**)

```dockerfile
FROM mcr.microsoft.com/dotnet/sdk:10.0 AS build

WORKDIR /src

COPY ["Api/Api.csproj", "Api/"]
COPY ["Domain/Domain.csproj", "Domain/"]
COPY ["Service/Service.csproj", "Service/"]
RUN dotnet restore "Api/Api.csproj"

COPY . .
RUN dotnet publish "Api/Api.csproj" -c Release -o /app/publish

FROM mcr.microsoft.com/dotnet/aspnet:10.0
WORKDIR /app
COPY --from=build /app/publish .

EXPOSE 8080
ENV ASPNETCORE_URLS=http://+:8080

ENTRYPOINT ["dotnet", "Api.dll"]
```

**Este Dockerfile tem duas etapas (multi-stage build):**

### Etapa 1 - build

| Linha | O que faz |
|---|---|
| `FROM ...sdk:10.0 AS build` | Parte de uma imagem que tem o **SDK** (compilador). Dá o apelido `build` a esta etapa |
| `WORKDIR /src` | Define a pasta de trabalho dentro do container |
| `COPY [...csproj...]` | Copia **só os `.csproj`**, ainda não o código |
| `RUN dotnet restore` | Baixa as dependências |
| `COPY . .` | **Agora sim** copia o código-fonte |
| `RUN dotnet publish` | Compila em modo Release e joga o resultado em `/app/publish` |

> **Por que copiar os `.csproj` antes do código?** Cache. O Docker guarda o resultado de cada
> linha. Se você mudar só o código C#, as linhas de `COPY *.csproj` e `RUN dotnet restore`
> ficam idênticas, o Docker reaproveita e **pula o download das dependências**. Isso corta
> minutos de cada build. Se copiasse tudo de uma vez, qualquer mudança invalidaria o restore.

### Etapa 2 - runtime

| Linha | O que faz |
|---|---|
| `FROM ...aspnet:10.0` | Começa **do zero** com uma imagem que só tem o runtime, sem compilador |
| `COPY --from=build /app/publish .` | Traz **apenas o resultado compilado** da etapa 1 |
| `EXPOSE 8080` | Documenta que o container escuta na porta 8080 |
| `ENV ASPNETCORE_URLS=http://+:8080` | Manda o ASP.NET escutar nessa porta, em qualquer interface (`+`) |
| `ENTRYPOINT ["dotnet", "Api.dll"]` | O comando executado quando o container sobe |

> **Por que duas etapas?** A imagem do SDK tem ~800 MB. A do runtime, ~220 MB. Como a etapa 2
> começa com um `FROM` novo, **tudo da etapa 1 é descartado**, só o que você copiar
> explicitamente sobrevive. Resultado: imagem menor, sobe mais rápido, e não leva o compilador
> nem o código-fonte para o servidor.

## 5.3 Criar o .dockerignore

> **CRIAR** - `.dockerignore` (na raiz, ao lado do Dockerfile)

```gitignore
**/bin/
**/obj/
**/.vs/
**/.vscode/
**/.idea/
.git/
.gitignore
.env
**/*.user
```

**Por que isso é obrigatório:** o `COPY . .` do Dockerfile copia **tudo** da pasta. Sem esse
arquivo, as pastas `bin/` e `obj/` compiladas na sua máquina Windows entram no container Linux.
O arquivo `obj/project.assets.json` guarda **caminhos absolutos** (`C:\Users\...`) que não
existem dentro do container, e o build quebra, ou pior, usa binário velho e você fica
depurando um bug que já corrigiu.

## 5.4 Criar o docker-compose.yml

> **CRIAR** - `docker-compose.yml` (na raiz)

```yaml
services:
  api:
    build:
      context: .
      dockerfile: Dockerfile
    container_name: join-api
    ports:
      - "8080:8080"
```

**O que é:** o Dockerfile diz *como construir*. O Compose diz *como executar*, portas,
variáveis, volumes. Em vez de decorar um `docker run` gigante, você escreve uma vez e roda
`docker compose up`.

| Chave | Significado |
|---|---|
| `build.context: .` | O build usa a pasta atual como contexto |
| `container_name` | Nome fixo do container, em vez de um gerado aleatoriamente |
| `ports: "8080:8080"` | Mapeia porta **8080 da sua máquina** → **8080 do container** |

## 5.5 Abrir o terminal do WSL pelo VS Code

**Aqui o terminal muda.** Tudo até agora rodou no PowerShell. O `docker` roda no **Ubuntu
(WSL)**, e você vai abrir esse terminal **de dentro do próprio VS Code**, sem sair do editor.

Passo a passo:

1. Se o painel do terminal estiver fechado, abra: menu **Terminal → New Terminal**
2. No canto **superior direito do painel do terminal**, logo ao lado do botão **`+`**, existe uma
   **setinha para baixo** (`⌄`)
3. Clique nessa setinha, abre a lista de perfis de terminal disponíveis
4. Selecione **`Ubuntu (WSL)`**

Um segundo terminal abre, agora com o prompt do Linux, algo como:

```text
seu-usuario@sua-maquina:/mnt/c/Users/SeuNome/Documents/Join/join-api$
```

> **Por que abrir o WSL pelo VS Code, e não pelo menu Iniciar:** porque assim ele **já abre
> dentro da pasta do projeto**. Olhe o final do prompt: `/join-api`. É exatamente a pasta onde
> estão o `Dockerfile` e o `docker-compose.yml`, e é de lá que o comando `docker compose`
> precisa ser executado, porque ele procura esses arquivos na pasta corrente.
>
> Se você abrisse o Ubuntu pelo menu Iniciar, cairia em `~` (a home do Linux, que é outro lugar)
> e teria que navegar até `/mnt/c/Users/.../join-api` na mão. O erro clássico desse caminho é
> `no configuration file provided: not found`, o Docker não achou o `docker-compose.yml` porque
> você estava na pasta errada.

**O que é `/mnt/c/`:** é como o Linux enxerga o disco `C:` do Windows. Mesmos arquivos, caminho
diferente. Você continua editando tudo pelo VS Code normalmente, do lado Windows, os dois
enxergam os mesmos arquivos.

**Confirme que está no lugar certo antes de continuar:**

> **Terminal WSL (Ubuntu)**

```bash
pwd
ls Dockerfile
```

O `pwd` deve terminar em `/join-api`, e o `ls` deve imprimir `Dockerfile`. Se responder
`No such file or directory`, você está na pasta errada, feche esse terminal e abra de novo pela
setinha `⌄`.

> **Os dois terminais ficam abertos ao mesmo tempo.** A lista à direita do painel mostra
> `powershell` e `Ubuntu (WSL)`; clique para alternar entre eles. Daqui em diante: PowerShell
> para `git` e `dotnet`, Ubuntu para `docker`.

## 5.6 Testar com Docker

> **Terminal WSL (Ubuntu)**

```bash
docker compose up --build
```

O `--build` força reconstruir a imagem. Na primeira vez demora, o Docker está baixando as
imagens base do .NET.

Quando aparecer `Now listening on: http://[::]:8080`, abra:
**`http://localhost:8080/swagger`**

Teste o `GET /Filmes` de novo. Deve retornar `"Deu certo!"`.

> O container roda dentro do WSL, mas a porta `8080` é publicada na sua máquina Windows, por
> isso `localhost:8080` funciona no navegador normalmente.

Para derrubar: `Ctrl+C`, depois:

> **Terminal WSL (Ubuntu)**

```bash
docker compose down
```

> **Esse é o teste que importa.** Rodar com `dotnet run` prova que o código compila. Rodar em
> container prova que ele **funciona empacotado**, que é como vai para o servidor.

## 5.7 Enviar para o GitHub

Volte para a aba do **PowerShell** no painel do terminal, `git` é comando do Windows.

> **Terminal do VS Code (PowerShell)**

```powershell
git add .
git commit -m 'feat: projeto inicial'
git push
```

**Confira o que foi enviado:** o `git status` não deve listar `bin/`, `obj/` nem `.env`. Se
listar, o `.gitignore` não está funcionando.

## 5.8 Abrir os Pull Requests

Um PR é um pedido de revisão antes de juntar o código. É também **o gatilho do deploy**, em
breve.

### PR 1 - feature → develop

1. No GitHub, abra o repositório `join-api`
2. Aparece um aviso da branch recém-enviada → **Compare & pull request**
3. Confira: **base:** `develop` ← **compare:** `feature/{id-card}-projeto-base`
4. Título: `feat: projeto inicial`
5. **Create pull request** → **Merge pull request** → **Confirm merge**

### PR 2 - develop → main

1. **Pull requests** → **New pull request**
2. **base:** `main` ← **compare:** `develop`
3. **Create pull request** → **Merge pull request** → **Confirm merge**

Mova o card "Projeto base" para **Concluído**.

### Checkpoint 5

- [ ] Você sabe abrir o terminal **Ubuntu (WSL)** pelo VS Code, e ele abre em `/join-api`
- [ ] `docker compose up --build` sobe e `/swagger` responde na 8080
- [ ] Código commitado e enviado
- [ ] `bin/`, `obj/` e `.env` **não** estão no GitHub
- [ ] `develop` e `main` atualizadas com o projeto base

---

# PARTE 6 - Primeiro deploy no Render

O Render conecta no seu repositório, detecta o Dockerfile, constrói a imagem e sobe o
container. A cada push na branch configurada, ele repete tudo sozinho. Isso é **CD**,
Continuous Deployment.

## 6.1 Criar a conta

Acesse <https://dashboard.render.com> e cadastre-se **com o GitHub**. Autorize o acesso ao
repositório `join-api` quando for solicitado.

## 6.2 Criar o serviço de homologação

1. Acesse <https://dashboard.render.com/web/new>
2. **Connect GitHub** e selecione o repositório `join-api`
3. Preencha:

| Campo | Valor |
|---|---|
| **Name** | `hml-join-api` |
| **Language** | `Docker` - deve ser detectado sozinho pelo Dockerfile |
| **Branch** | `develop` |
| **Region** | qualquer uma - **anote, use a mesma no próximo serviço** |
| **Instance Type** | **Free** |

4. **Deploy Web Service**

O prefixo `hml-` no nome é o que diferencia visualmente os dois serviços no painel.
A branch `develop` é o que faz este ambiente receber o que ainda está em teste.

## 6.3 Criar o serviço de produção

Repita o processo:

| Campo | Valor |
|---|---|
| **Name** | `join-api` |
| **Language** | `Docker` |
| **Branch** | `main` |
| **Region** | a mesma do anterior |
| **Instance Type** | **Free** |

## 6.4 Acompanhar o deploy

Em cada serviço, abra a aba **Logs**. Você verá o Render executando as etapas do Dockerfile.

> **Free tier:** o serviço hiberna após ~15 min sem acesso. A primeira requisição depois
> disso leva **30 a 60 segundos**. Não é erro.

**Agora observe os logs.** Algo deu errado. Continue para a Parte 7.

### Checkpoint 6

- [ ] `hml-join-api` criado, branch `develop`, plano Free
- [ ] `join-api` criado, branch `main`, plano Free
- [ ] Você abriu a aba **Logs** e viu que o deploy falhou

---

# PARTE 7 - O deploy quebrou (hotfix)

## 7.1 Ler o erro

Nos **Logs** do Render, o build passou, mas na execução aparece:

```
Unhandled exception. System.IO.IOException: The configured user limit (128) on the number
of inotify instances has been reached, or the per-process limit on the number of open file
descriptors has been reached.
   at System.IO.FileSystemWatcher.StartRaisingEvents()
   at Microsoft.Extensions.FileProviders.Physical.PhysicalFilesWatcher.TryEnableFileSystemWatcher()
   at Microsoft.Extensions.Configuration.Json.JsonConfigurationSource.Build(IConfigurationBuilder builder)
   at Microsoft.AspNetCore.Builder.WebApplication.CreateBuilder(String[] args)
   at Program.<Main>$(String[] args)
```

O serviço entra em loop de reinício e nunca responde.

> **Primeira lição de leitura de log:** separe **build** de **runtime**. O build compilou sem
> erro. A falha é na execução. Se você ficar procurando problema no Dockerfile de compilação,
> vai perder tempo.

## 7.2 Entender a causa

Leia o stack trace **de baixo para cima**, ele conta a história na ordem inversa:

```
Program.<Main>$                          ← seu código começou
  WebApplication.CreateBuilder           ← montando a aplicação
    JsonConfigurationSource.Build        ← carregando appsettings.json
      PhysicalFilesWatcher                ← criando um "vigia" do arquivo
        FileSystemWatcher.StartRaising    ← e aqui estourou
```

**O que está acontecendo:**

`WebApplication.CreateBuilder()` carrega o `appsettings.json` com a opção
`reloadOnChange: true` ligada por padrão. Isso cria um **FileSystemWatcher**, um vigia que
monitora o arquivo para recarregar a configuração se alguém editá-lo em tempo de execução.

No Linux, esse vigia consome uma **instância de inotify** do kernel. O kernel limita quantas
cada usuário pode abrir (`fs.inotify.max_user_instances`).

Na sua máquina esse limite é folgado, **8192**. Nos servidores do Render, onde **centenas de
containers dividem o mesmo kernel**, ele esgota. Quando esgota, o vigia lança `IOException`.

E como isso acontece **dentro do `CreateBuilder`**, antes de qualquer `try/catch` do seu
código, o processo morre no boot.

> **Por isso funcionou na sua máquina e quebrou no servidor.** Este é um bug de ambiente:
> não reproduz em `docker compose up` local porque sua máquina tem inotify sobrando. Só aparece
> sob a pressão de recurso de um host compartilhado. É exatamente o tipo de problema que
> justifica ter um ambiente de homologação.

**A correção:** desligar o vigia. Em container isso não custa nada, o `appsettings.json`
nunca muda durante a execução, então recarregá-lo é inútil.

## 7.3 Criar o card do incidente

No Trello, crie um card na coluna **Em andamento**:

**`[Hotfix] Falha no deploy`**

Anote o número. Incidente também é tarefa, e também é rastreado.

## 7.4 Criar a branch de hotfix

**A partir da `main`** - porque é produção que está quebrada:

> **Terminal do VS Code (PowerShell)**

```powershell
git fetch
git checkout main
git pull
git checkout -b hotfix/{id-card}-falha-deploy
```

| Comando | Por quê |
|---|---|
| `git fetch` | Atualiza a lista de branches remotas |
| `git checkout main` + `git pull` | Garante que você parte do estado **real** de produção |
| `git checkout -b hotfix/...` | Cria a branch de correção |

> **Por que da `main` e não da `develop`?** Porque `develop` pode ter código ainda não testado.
> Um hotfix precisa levar para produção **só a correção**, nada mais.

## 7.5 Corrigir o Dockerfile

> **EDITAR** - `Dockerfile`
> Adicione as duas linhas `ENV` **na segunda etapa**, logo depois do `COPY --from=build`:

```dockerfile
FROM mcr.microsoft.com/dotnet/aspnet:10.0
WORKDIR /app
COPY --from=build /app/publish .

# Desabilita file watchers ANTES do app iniciar
ENV DOTNET_hostBuilder__reloadConfigOnChange=false
ENV DOTNET_EnableDiagnostics=0

EXPOSE 8080
ENV ASPNETCORE_URLS=http://+:8080

ENTRYPOINT ["dotnet", "Api.dll"]
```

| Variável | O que faz |
|---|---|
| `DOTNET_hostBuilder__reloadConfigOnChange=false` | **Esta é a correção.** Desliga o FileSystemWatcher da configuração - nenhum inotify é consumido |
| `DOTNET_EnableDiagnostics=0` | Desliga o socket de diagnóstico do .NET. Não corrige este bug, mas reduz consumo e superfície de ataque em produção |

> **As duas `ENV` precisam estar na segunda etapa.** Variáveis declaradas na etapa `build`
> são descartadas no `FROM` seguinte, junto com o resto. Não guarde "linha 19", guarde
> **"depois do `COPY --from=build`"**.

## 7.6 Corrigir o Program.cs

> **EDITAR** - `Api/Program.cs`
> **Remova** esta linha:

```csharp
app.UseHttpsRedirection();
```

O arquivo fica assim:

```csharp
var builder = WebApplication.CreateBuilder(args);

builder.Services.AddControllers();
builder.Services.AddEndpointsApiExplorer();
builder.Services.AddSwaggerGen();

var app = builder.Build();

app.UseSwagger();
app.UseSwaggerUI();

app.UseAuthorization();

app.MapControllers();

app.Run();
```

**Por quê:** o Render termina o HTTPS **no proxy dele** e entrega HTTP puro ao seu container.
Mandar a aplicação redirecionar para HTTPS ali dentro é redundante - e, dependendo da
configuração, causa loop de redirecionamento. Aplicação atrás de proxy que já cuida do TLS não
deve fazer esse redirect.

## 7.7 Enviar a correção

> **Terminal do VS Code (PowerShell)**

```powershell
git add .
git commit -m 'fix: Correção no Dockerfile para o deploy'
git push -u origin hotfix/{id-card}-falha-deploy
```

## 7.8 Os dois PRs do hotfix

Aqui está a regra do GitFlow que mais se erra. Um hotfix precisa de **dois merges**.

### PR 1 - hotfix → main (restaura produção)

1. **New pull request**
2. **base:** `main` ← **compare:** `hotfix/{id-card}-falha-deploy`
3. **Create** → **Merge** → **Confirm**

### PR 2 - hotfix → develop (impede que o bug volte)

1. **New pull request**
2. **base:** `develop` ← **compare:** `hotfix/{id-card}-falha-deploy`
3. **Create** → **Merge** → **Confirm**

> **Se você fizer só o primeiro:** produção fica consertada agora. Mas `develop` continua com o
> Dockerfile antigo. No próximo merge `develop → main`, **o bug volta para produção**, e você
> vai passar horas se perguntando como um bug corrigido reapareceu. Faça sempre os dois.

## 7.9 Validar os dois ambientes

Cada merge disparou um deploy. Acompanhe os **Logs** dos dois serviços até ficarem verdes.

Depois acesse no navegador:

- **HML:** `https://hml-join-api.onrender.com/swagger`
- **PROD:** `https://join-api.onrender.com/swagger`

> As URLs reais estão no topo de cada serviço no painel do Render. Se o Render tiver adicionado
> um sufixo aleatório ao nome, use a URL que ele mostra.

Em ambos, teste o `GET /Filmes`. Deve retornar `"Deu certo!"`.

Mova o card do hotfix para **Concluído**.

### Checkpoint 7

- [ ] Dockerfile com as duas `ENV` **na etapa runtime**
- [ ] `app.UseHttpsRedirection()` removido
- [ ] PR hotfix → `main` mergeado
- [ ] PR hotfix → `develop` mergeado
- [ ] `/swagger` abre e responde nos **dois** ambientes

**Você tem dois ambientes no ar.** Agora vamos colocar uma API de verdade neles.

---

# PARTE 8 - Feature: listar filmes

Nesta parte a API deixa de retornar texto fixo e passa a buscar dados de uma fonte externa.
São muitos arquivos, siga na ordem, cada um explica o próximo.

## 8.1 Card e branch

Mova o card **"Criação da API de listar filmes"** para *Em andamento* e anote o número.

> **Terminal do VS Code (PowerShell)**

```powershell
git fetch
git checkout develop
git pull
git checkout -b feature/{id-card}-criacao-api-listar-filmes
git push -u origin feature/{id-card}-criacao-api-listar-filmes
```

> **Sempre `git pull` na `develop` antes de criar a branch.** Sua `develop` local está
> desatualizada, o hotfix foi mergeado pelo GitHub, não pela sua máquina.

## 8.2 A fonte de dados

Os filmes vêm deste JSON:

```
https://raw.githubusercontent.com/join-unifenas/filmes/main/db-hml.json
```

Estrutura do arquivo:

```json
{
    "filmes": [
        {
            "id": 1,
            "titulo": "Batman: A Queda do Morcego - Parte 1",
            "lingua_original": "en",
            "titulo_original": "Batman: Knightfall Part 1",
            "descricao": "O Asilo Arkham foi destruído...",
            "painel": "https://image.tmdb.org/t/p/w500/caBIy...jpg",
            "poster": "https://image.tmdb.org/t/p/w500/qJJTW...jpg",
            "ano": 2026
        }
    ]
}
```

Repare: os filmes **não** estão na raiz, estão dentro de uma propriedade `filmes`. Nossas
classes precisam refletir isso.

Existe uma segunda URL, `db.json`, com **filmes diferentes**. É ela que vamos usar em produção.

## 8.3 Criar o DTO do filme

> **CRIAR** - `Domain/Dtos/FilmeDto.cs`
> (crie a pasta `Dtos` dentro de `Domain` primeiro)

```csharp
namespace Domain.Dtos;

public class FilmeDto
{
    public int Id { get; set; }
    public string Titulo { get; set; } = string.Empty;
    public string Descricao { get; set; } = string.Empty;
    public int Ano { get; set; }
    public string Poster { get; set; } = string.Empty;
}
```

**O que é um DTO:** *Data Transfer Object*, uma classe cujo único papel é carregar dados entre
camadas. Sem lógica, só propriedades.

**Por que só 5 dos 8 campos?** Porque `lingua_original`, `titulo_original` e `painel` não
interessam à nossa API. O que você não declara é simplesmente ignorado na conversão. DTO é um
**contrato do que você expõe**, não uma cópia da fonte.

> **Sobre o `= string.Empty;`:** o projeto tem *nullable reference types* ativado. Sem o
> inicializador, o compilador emite o aviso `CS8618` avisando que a propriedade pode ficar
> nula. Inicializar resolve e deixa o build limpo.

**E os nomes em maiúscula?** O JSON tem `titulo` (minúsculo) e a classe tem `Titulo`. Funciona:
o conversor do RestSharp faz a correspondência **ignorando maiúsculas e minúsculas**.

## 8.4 Criar o DTO da resposta

> **CRIAR** - `Domain/Dtos/FilmesResponse.cs`

```csharp
namespace Domain.Dtos;

public class FilmesResponse
{
    public List<FilmeDto> Filmes { get; set; } = [];
}
```

**Por que uma segunda classe?** Porque o JSON é `{ "filmes": [...] }`, não `[...]` direto.
Precisamos de uma classe que represente o **envelope**, com uma propriedade `Filmes` que casa
com a chave do JSON. Sem ela, o conversor não saberia onde encontrar a lista.

## 8.5 Criar a interface do serviço

> **CRIAR** - `Service/Interface/IFilmesService.cs`
> (crie a pasta `Interface` dentro de `Service`)

```csharp
using Domain.Dtos;

namespace Service.Interface;

public interface IFilmesService
{
    Task<List<FilmeDto>> ObterFilmes();
}
```

**Por que uma interface antes da implementação:** a interface é o **contrato**, diz *o que* o
serviço faz, sem dizer *como*. O controller vai depender da interface, não da classe concreta.

Isso permite trocar a implementação (buscar de um banco, de um cache, de um mock em testes) sem
tocar no controller. É a inversão de dependência: quem usa não conhece quem faz.

`Task<...>` indica método **assíncrono**: chamadas de rede não travam a thread enquanto esperam
resposta.

## 8.6 Instalar o RestSharp

> **Terminal do VS Code (PowerShell)**

```powershell
dotnet add ./Service/Service.csproj package RestSharp
```

**O que é:** uma biblioteca que simplifica chamadas HTTP. Ela faz a requisição **e** converte o
JSON da resposta direto para suas classes C#, em uma linha.

Note que instalamos no **Service**, não no Api. Quem fala com a fonte externa é o Service, o
Api nem precisa saber que existe HTTP envolvido.

## 8.7 Implementar o serviço

> **CRIAR** - `Service/FilmesService.cs`

```csharp
using Service.Interface;
using RestSharp;
using Domain.Dtos;

namespace Service;

public class FilmesService : IFilmesService
{
    private readonly RestClient _client;

    public FilmesService(RestClient client)
    {
        _client = client;
    }

    public async Task<List<FilmeDto>> ObterFilmes()
    {
        var request = new RestRequest("", Method.Get);
        var response = await _client.ExecuteAsync<FilmesResponse>(request);

        if (!response.IsSuccessful)
            throw new Exception($"Erro ao obter filmes: {response.ErrorMessage}");

        return response.Data?.Filmes ?? new List<FilmeDto>();
    }
}
```

**Parte por parte:**

```csharp
private readonly RestClient _client;

public FilmesService(RestClient client)
{
    _client = client;
}
```
**Injeção de dependência por construtor.** A classe **não cria** o `RestClient`, ela o recebe
pronto. Quem monta e entrega é o container de DI, configurado no `Program.cs` (seção 8.11).
Vantagem: a URL base fica em um lugar só, e em testes você injeta um cliente falso.

```csharp
var request = new RestRequest("", Method.Get);
```
Cria a requisição. A rota é `""` porque a URL completa já está no `RestClient`, não há caminho
adicional a acrescentar.

```csharp
var response = await _client.ExecuteAsync<FilmesResponse>(request);
```
Executa e **já converte** o JSON para `FilmesResponse`. O `<FilmesResponse>` é o que diz ao
RestSharp qual classe usar. O `await` libera a thread enquanto a rede responde.

```csharp
if (!response.IsSuccessful)
    throw new Exception($"Erro ao obter filmes: {response.ErrorMessage}");
```
Se a fonte externa falhou (404, timeout, fora do ar), lançamos exceção. Quem trata é o
controller.

```csharp
return response.Data?.Filmes ?? new List<FilmeDto>();
```
Dois operadores de segurança:
- `?.` - se `Data` for nulo, não tenta acessar `.Filmes` (evita `NullReferenceException`)
- `??` - se o resultado for nulo, devolve lista vazia

Assim o método **nunca retorna null**. Quem chama não precisa verificar.

## 8.8 Atualizar o controller

> **EDITAR** - `Api/Controllers/FilmesController.cs`
> Substitua **todo** o conteúdo:

```csharp
using Microsoft.AspNetCore.Mvc;
using Service.Interface;

namespace Api.Controllers;

[ApiController]
[Route("[controller]")]
public class FilmesController : ControllerBase
{
    private readonly IFilmesService _filmesService;

    public FilmesController(IFilmesService filmesService)
    {
        _filmesService = filmesService;
    }

    [HttpGet]
    public async Task<IActionResult> Get()
    {
        try
        {
            var filmes = await _filmesService.ObterFilmes();
            return Ok(filmes);
        }
        catch (Exception ex)
        {
            return StatusCode(500, $"Erro ao obter os filmes: {ex.Message}");
        }
    }
}
```

**O que mudou:**

| Antes | Depois |
|---|---|
| Sem construtor | Recebe `IFilmesService` por injeção |
| `IActionResult Get()` | `async Task<IActionResult> Get()` |
| `Ok("Deu certo!")` | `Ok(filmes)` com `try/catch` |

**Repare no tipo injetado:** `IFilmesService`, a **interface**, não `FilmesService`. O
controller não sabe qual implementação está recebendo. É isso que torna a peça substituível.

**O `try/catch`:** o serviço lança exceção quando a fonte falha. Sem tratamento, o ASP.NET
devolveria uma página de erro genérica. Com ele, devolvemos um `500` com mensagem clara.

## 8.9 Criar o arquivo .env

Até agora a URL da fonte não aparece em lugar nenhum. Ela **não pode ficar fixa no código**,
HML e PROD usam URLs diferentes, e o mesmo código roda nos dois.

> **CRIAR** - `.env` (na raiz de `join-api/`)

```env
FILMES_API_URL=https://raw.githubusercontent.com/join-unifenas/filmes/main/db-hml.json
```

> **Este arquivo não vai para o GitHub.** O `.gitignore` do .NET já ignora `.env`, e é
> assim que deve ser: arquivos de ambiente costumam guardar segredos.
>
> **Consequência importante:** o Render **nunca vai receber este arquivo**. Vamos cadastrar a
> variável direto no painel dele, na seção 8.13. Se pular esse passo, o container sobe e morre.

**Confirme que está ignorado:**

> **Terminal do VS Code (PowerShell)**

```powershell
git status
```

O `.env` **não** deve aparecer na lista.

## 8.10 Instalar o DotNetEnv

> **Terminal do VS Code (PowerShell)**

```powershell
dotnet add ./Api/Api.csproj package DotNetEnv
```

**Para quê:** o .NET lê variáveis de ambiente do sistema, mas não lê arquivos `.env`. Esta
biblioteca lê o arquivo e injeta o conteúdo nas variáveis do processo, **só para o
desenvolvimento local**. No servidor, as variáveis já vêm do ambiente.

## 8.11 Configurar o Program.cs

> **EDITAR** - `Api/Program.cs`
> Substitua **todo** o conteúdo:

```csharp
using RestSharp;
using Service;
using Service.Interface;

DotNetEnv.Env.Load("../.env");

var builder = WebApplication.CreateBuilder(args);

builder.Services.AddControllers();
builder.Services.AddEndpointsApiExplorer();
builder.Services.AddSwaggerGen();

var filmesApiUrl = builder.Configuration["FILMES_API_URL"];

builder.Services.AddSingleton(
    new RestClient(filmesApiUrl!)
);

builder.Services.AddScoped<IFilmesService, FilmesService>();

var app = builder.Build();

app.UseSwagger();
app.UseSwaggerUI();

app.UseAuthorization();

app.MapControllers();

app.Run();
```

**As linhas novas:**

```csharp
DotNetEnv.Env.Load("../.env");
```
Lê o `.env` e injeta nas variáveis do processo. **Precisa vir antes do `CreateBuilder`**, o
builder lê as variáveis de ambiente no momento em que é criado. Se inverter a ordem, a variável
ainda não existe e tudo quebra.

O caminho `"../.env"` é relativo à pasta de execução. Como você roda de dentro de `Api/`,
`../` chega na raiz do repositório. No container o arquivo não existe, a biblioteca
simplesmente não encontra e segue sem fazer nada, o que é exatamente o comportamento desejado.

```csharp
var filmesApiUrl = builder.Configuration["FILMES_API_URL"];
```
Lê a variável. **Duas origens, mesmo código:** local, veio do `.env`; no Render, veio do painel.

```csharp
builder.Services.AddSingleton(new RestClient(filmesApiUrl!));
```
Registra o `RestClient` como **Singleton**, uma única instância para toda a aplicação. É o
correto para clientes HTTP: criar um por requisição esgota as conexões do sistema operacional.

```csharp
builder.Services.AddScoped<IFilmesService, FilmesService>();
```
A linha que **liga a interface à implementação**. Traduzindo: *"quando alguém pedir um
`IFilmesService`, entregue um `FilmesService`"*. É por isso que o construtor do controller
funciona, o container sabe o que injetar.

`Scoped` = uma instância por requisição HTTP.

| Ciclo de vida | Quando usar |
|---|---|
| `Singleton` | Uma para a aplicação inteira - clientes HTTP, cache |
| `Scoped` | Uma por requisição - serviços, repositórios |
| `Transient` | Uma nova a cada injeção |

## 8.12 Atualizar o docker-compose.yml

> **EDITAR** - `docker-compose.yml`
> Adicione o bloco `environment` no final:

```yaml
services:
  api:
    build:
      context: .
      dockerfile: Dockerfile
    container_name: join-api
    ports:
      - "8080:8080"
    environment:
      FILMES_API_URL: ${FILMES_API_URL}
```

O Compose lê o `.env` da mesma pasta **automaticamente** e substitui `${FILMES_API_URL}` pelo
valor. Assim o container recebe a variável sem que ela esteja escrita no arquivo.

## 8.13 Testar local

Primeiro sem Docker:

> **Terminal do VS Code (PowerShell)**

```powershell
cd Api
dotnet run
```

Abra `http://localhost:5142/swagger` e execute o `GET /Filmes`. Deve vir a lista de filmes em
JSON.

`Ctrl+C` e volte para a raiz:

> **Terminal do VS Code (PowerShell)**

```powershell
cd ..
```

Agora teste em container. **Troque para a aba `Ubuntu (WSL)`** no painel do terminal, se você
fechou aquele terminal, reabra pela setinha `⌄` ao lado do `+` (seção 5.5):

> **Terminal WSL (Ubuntu)**

```bash
docker compose up --build
```

Abra `http://localhost:8080/swagger` e teste de novo.

> **Se der erro `ArgumentNullException (Parameter 'baseUrl')`:** a variável não foi lida.
> Verifique se o `.env` existe na raiz, se o nome está exatamente `FILMES_API_URL`, e se você
> rodou o `dotnet run` de dentro da pasta `Api/`.

Derrube com `docker compose down`, no mesmo terminal do Ubuntu.

## 8.14 Enviar o código

> **Terminal do VS Code (PowerShell)**

```powershell
git add .
git commit -m 'feat: Criação da API de listar filmes'
git push
```

## 8.15 Cadastrar as variáveis no Render - **antes do merge**

> **Faça este passo agora, antes de abrir o PR.** No próximo merge o deploy dispara
> automaticamente. Se a variável não estiver cadastrada, o container sobe, não encontra a URL e
> **morre no boot**. Cadastre primeiro; o deploy já nasce funcionando.

### No serviço de homologação

1. Render → serviço **`hml-join-api`** → aba **Environment**
2. **Add Environment Variable**
3. Preencha:

| Campo | Valor |
|---|---|
| **Key** | `FILMES_API_URL` |
| **Value** | `https://raw.githubusercontent.com/join-unifenas/filmes/main/db-hml.json` |

4. **Save Changes**

### No serviço de produção

1. Render → serviço **`join-api`** → aba **Environment**
2. **Add Environment Variable**

| Campo | Valor |
|---|---|
| **Key** | `FILMES_API_URL` |
| **Value** | `https://raw.githubusercontent.com/join-unifenas/filmes/main/db.json` |

3. **Save Changes**

> **Repare na diferença:** `db-hml.json` em homologação, `db.json` em produção. São arquivos
> com filmes diferentes. É assim que simulamos dois bancos de dados, o mesmo código, dados
> distintos, definidos por configuração e não por código.

Salvar dispara um redeploy automático. Aguarde ficar verde.

## 8.16 PR para develop e validação em HML

1. **New pull request** → **base:** `develop` ← **compare:** `feature/{id-card}-criacao-api-listar-filmes`
2. **Create** → **Merge** → **Confirm**
3. Acompanhe os **Logs** de `hml-join-api`
4. Acesse `https://hml-join-api.onrender.com/swagger` e execute o `GET /Filmes`

Deve retornar a lista de filmes.

## 8.17 PR para main e validação em PROD

Só depois de validar em HML:

1. **New pull request** → **base:** `main` ← **compare:** `develop`
2. **Create** → **Merge** → **Confirm**
3. Acesse `https://join-api.onrender.com/swagger` e execute o `GET /Filmes`

> **O teste que prova que tudo funcionou:** compare o primeiro filme dos dois ambientes. Os
> títulos devem ser **diferentes**, HML lê `db-hml.json`, PROD lê `db.json`. Se vierem iguais,
> alguma variável foi cadastrada errada.

Mova o card para **Concluído**.

### Checkpoint 8

- [ ] `GET /filmes` retorna a lista local (com e sem Docker)
- [ ] `FILMES_API_URL` cadastrada nos **dois** serviços, com valores **diferentes**
- [ ] HML respondendo com dados de `db-hml.json`
- [ ] PROD respondendo com dados de `db.json`
- [ ] Os dois ambientes retornam filmes **diferentes**

---

# PARTE 9 - Feature: obter filme por ID

Agora o fluxo se repete. Esta parte é mais curta de propósito, você já conhece o caminho.

## 9.1 Card e branch

Mova o card **"Criação da API de obter filme por ID"** para *Em andamento*.

> **Terminal do VS Code (PowerShell)**

```powershell
git fetch
git checkout develop
git pull
git checkout -b feature/{id-card}-criacao-api-obter-filme-por-id
git push -u origin feature/{id-card}-criacao-api-obter-filme-por-id
```

## 9.2 Adicionar o método ao contrato

> **EDITAR** - `Service/Interface/IFilmesService.cs`
> Adicione a segunda linha dentro da interface:

```csharp
using Domain.Dtos;

namespace Service.Interface;

public interface IFilmesService
{
    Task<List<FilmeDto>> ObterFilmes();
    Task<FilmeDto?> ObterFilmePorID(int id);
}
```

**Sempre a interface primeiro.** Você define o contrato, depois cumpre. Assim que salvar, o
projeto **para de compilar**, `FilmesService` prometeu implementar `IFilmesService` e agora
está devendo um método. O compilador virou sua lista de tarefas.

**Repare no `FilmeDto?`** - com interrogação. Significa que o retorno **pode ser nulo**: se o ID
não existir, não há filme. O tipo comunica isso a quem chama.

## 9.3 Implementar no serviço

> **EDITAR** - `Service/FilmesService.cs`
> Adicione o método **depois** de `ObterFilmes()`, antes da última chave:

```csharp
public async Task<FilmeDto?> ObterFilmePorID(int id)
{
    var request = new RestRequest("", Method.Get);
    var response = await _client.ExecuteAsync<FilmesResponse>(request);

    if (!response.IsSuccessful)
        throw new Exception($"Erro ao obter filmes: {response.ErrorMessage}");

    return response.Data?.Filmes?.FirstOrDefault(f => f.Id == id);
}
```

Quase idêntico ao anterior. A diferença é a última linha:

```csharp
return response.Data?.Filmes?.FirstOrDefault(f => f.Id == id);
```

`FirstOrDefault` percorre a lista e devolve **o primeiro item que satisfaz a condição**. Se
nenhum satisfizer, devolve `null` (o "default" de um tipo de referência), em vez de lançar
exceção, como faria o `First`.

`f => f.Id == id` é uma expressão lambda: *"para cada filme `f`, teste se o `Id` dele é igual
ao `id` procurado"*.

> **Observação honesta:** este método baixa o JSON **inteiro** só para achar um filme. Em uma
> API com banco de dados você faria uma consulta filtrada. Aqui a fonte é um arquivo estático,
> não há como pedir "só o filme 3". É uma limitação da fonte, não um erro de código.

## 9.4 Adicionar o endpoint no controller

> **EDITAR** - `Api/Controllers/FilmesController.cs`
> Adicione o método **depois** do `Get()` existente:

```csharp
[HttpGet("{id}")]
public async Task<IActionResult> Get(int id)
{
    try
    {
        var filme = await _filmesService.ObterFilmePorID(id);
        if (filme == null)
            return NotFound("Filme não encontrado");

        return Ok(filme);
    }
    catch (Exception ex)
    {
        return StatusCode(500, $"Erro ao obter o filme: {ex.Message}");
    }
}
```

**Pontos importantes:**

| Trecho | Explicação |
|---|---|
| `[HttpGet("{id}")]` | Acrescenta `/{id}` à rota → `/filmes/3`. O `{id}` é um **parâmetro de rota** |
| `Get(int id)` | O ASP.NET pega o valor da URL e passa como argumento, convertendo para `int` automaticamente |
| Dois métodos `Get` | Sobrecarga: o ASP.NET distingue pela rota. `/filmes` → o primeiro; `/filmes/3` → o segundo |
| `NotFound(...)` | Retorna **404**, o código correto para "não existe" |
| `StatusCode(500, ...)` | Retorna **500** - erro do servidor, coisa diferente de "não achei" |

> **Por que 404 e 500 são diferentes:** `404` diz *"sua requisição está certa, mas esse recurso
> não existe"*. `500` diz *"eu quebrei"*. Um cliente que recebe 404 sabe que não adianta tentar
> de novo; com 500, tentar de novo pode funcionar. Escolher o código certo é parte de projetar
> uma API.

## 9.5 Testar local

> **Terminal do VS Code (PowerShell)**

```powershell
cd Api
dotnet run
```

No Swagger, agora há **dois** endpoints. Teste os três casos:

| Requisição | Esperado |
|---|---|
| `GET /Filmes` | `200` + lista completa |
| `GET /Filmes/1` | `200` + um filme |
| `GET /Filmes/9999` | `404` + `Filme não encontrado` |

Teste também em container. Pare o `dotnet run` com `Ctrl+C` e volte para a raiz:

> **Terminal do VS Code (PowerShell)**

```powershell
cd ..
```

Troque para a aba **`Ubuntu (WSL)`**:

> **Terminal WSL (Ubuntu)**

```bash
docker compose up --build
```

Derrube com `docker compose down` quando terminar.

## 9.6 Enviar e publicar

> **Terminal do VS Code (PowerShell)**

```powershell
git add .
git commit -m 'feat: Criação da api para obter filme por ID'
git push
```

As variáveis de ambiente já estão cadastradas no Render, não precisa mexer.

### PR para develop

1. **base:** `develop` ← **compare:** `feature/{id-card}-criacao-api-obter-filme-por-id`
2. **Merge**
3. Valide em `https://hml-join-api.onrender.com/swagger`, teste `/Filmes/1` e `/Filmes/9999`

### PR para main

1. **base:** `main` ← **compare:** `develop`
2. **Merge**
3. Valide em `https://join-api.onrender.com/swagger`

Mova o card para **Concluído**.

### Checkpoint 9

- [ ] `GET /filmes/1` retorna um filme nos dois ambientes
- [ ] `GET /filmes/9999` retorna `404` nos dois ambientes
- [ ] Todos os cards em *Concluído*

---

# PARTE 10 - Encerramento

## 10.1 O que você construiu

Uma API .NET em três camadas, containerizada, rodando em dois ambientes na nuvem, com deploy
automático a cada merge, e você passou por um incidente de produção no meio do caminho.

## 10.2 O fluxo que você praticou

```
Card no Trello
     │
     ▼
Branch feature/{id}-descricao  ← a partir de develop, sempre atualizada
     │
     ▼
Código + teste local (dotnet run E docker compose up)
     │
     ▼
Configuração de ambiente cadastrada ANTES do merge
     │
     ▼
Push → PR para develop → DEPLOY AUTOMÁTICO EM HML
     │
     ▼
Validação em homologação
     │
     ▼
PR develop → main → DEPLOY AUTOMÁTICO EM PRODUÇÃO
     │
     ▼
Validação em produção → Card em Concluído
```

E, quando produção quebra:

```
Incidente → Card [Hotfix]
     │
     ▼
Branch hotfix/{id}  ← a partir da MAIN
     │
     ▼
Correção mínima
     │
     ├──► PR para main     (conserta produção agora)
     └──► PR para develop  (impede o bug de voltar)
```

## 10.3 As ideias que ficam

**Ambiente é configuração, não código.** O mesmo binário roda em HML e PROD. O que muda é uma
variável. Se você precisa recompilar para trocar de ambiente, algo está errado.

**Homologação existe para achar o que o local não acha.** O bug de inotify não reproduzia na
sua máquina. Só apareceu sob a pressão de um host compartilhado. É exatamente para isso que
serve um ambiente intermediário.

**Branch tem significado, não só nome.** `main` é o que está no ar. `develop` é o que vai
subir. `hotfix` é urgência que precisa de dois merges. Quando o time respeita o acordo, o
histórico do Git conta a verdade sobre o sistema.

**Container é o que torna "funciona na minha máquina" irrelevante.** O que você testou com
`docker compose up` é literalmente o que roda no servidor.

**Leia o stack trace de baixo para cima.** Ele conta a história na ordem inversa. E separe
sempre erro de build de erro de runtime, são problemas de naturezas diferentes.

## 10.4 Referência rápida de comandos

> **Terminal do VS Code (PowerShell)** - `git` e `dotnet`

```powershell
# Configuração inicial do Git (uma vez por máquina)
git config --global user.name "Seu Nome Completo"
git config --global user.email "{seu-email-institucional}"
git config --global init.defaultBranch main
git config --global core.autocrlf true
git config --global --list

# Nova feature
git fetch; git checkout develop; git pull
git checkout -b feature/{id}-descricao
git push -u origin feature/{id}-descricao

# Hotfix
git fetch; git checkout main; git pull
git checkout -b hotfix/{id}-descricao

# Rodar local
cd Api; dotnet run
cd ..

# Publicar
git add .
git commit -m 'feat: descrição'
git push
```

> **Por que `;` e não `&&`:** o PowerShell do Windows não aceita `&&` para encadear comandos.
> No PowerShell use `;`; no terminal do Ubuntu, `&&` funciona normalmente.

E do outro lado:

> **Terminal WSL (Ubuntu)** - `docker`

```bash
# Rodar em container (a partir da raiz de join-api/)
docker compose up --build
docker compose down

# Conferir que está na pasta certa
pwd
ls Dockerfile
```

## 10.5 Erros comuns

| Erro | Causa | Solução |
|---|---|---|
| `git: command not found` / `'git' não é reconhecido` | Git não instalado, ou instalado sem a opção de PATH | Reinstale marcando `Git from the command line...` (seção 1.5) |
| `Please tell me who you are` no commit | `user.name` / `user.email` não configurados | Rode os dois `git config --global` da seção 1.5 |
| Commits sem sua foto no GitHub | Commitou com e-mail diferente do da conta | Corrija com `git config --global user.email` e use o institucional |
| `docker: command not found` no PowerShell | Terminal errado | `docker` roda no **Ubuntu (WSL)** - troque de aba no painel do terminal |
| `dotnet: command not found` no Ubuntu | Terminal errado | `dotnet` roda no **PowerShell** - o SDK está instalado no Windows |
| `no configuration file provided: not found` | Terminal do WSL aberto na pasta errada | Abra o Ubuntu **pelo VS Code** (seção 5.5) e confira com `pwd` |
| `IOException: inotify instances` | Falta a `ENV` no Dockerfile | `DOTNET_hostBuilder__reloadConfigOnChange=false` na etapa runtime |
| `ArgumentNullException ('baseUrl')` | `FILMES_API_URL` não configurada | Cadastre no `.env` local ou no Render → Environment |
| `404` em `/` | Não existe rota raiz | Use `/filmes` ou `/swagger` |
| Primeira chamada demora 1 min | Free tier hibernou | Normal. Aguarde |
| HML e PROD com os mesmos dados | Variável igual nos dois | Corrija o valor no serviço errado |
| Build Docker falha com caminho `C:\` | Falta `.dockerignore` | Crie o arquivo (seção 5.3) |
| Push em `develop` não deploya | Serviço na branch errada | Render → Settings → Branch |
| Bug corrigido voltou | Hotfix só foi para `main` | Faça também o PR para `develop` |

---

*Mini curso DevOps - Join API*
