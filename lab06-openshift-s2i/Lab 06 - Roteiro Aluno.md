# Lab 06 - Fábrica de Software Cloud-Native e Git Privado no Red Hat OpenShift: Gitea Rootless e Pipelines S2I

* **Programa:** MBA em MultiCloud Strategy & Architecture
* **Ambiente / Plataforma:** Red Hat Academy (DO180 v4.14 / Red Hat OpenShift Container Platform 4.14)
* **Stack Técnica:** Red Hat OpenShift S2I, BuildConfigs, ImageStreams, Gitea Rootless (porta 3000), Security Context Constraints (SCC `restricted-v2`), OpenShift Routes, Node.js Builder
* **Duração Estimada:** 35 a 40 minutos

---

## 🎯 Objetivo do Lab

Capacitar o aluno a estruturar uma fábrica de software corporativa soberana e automatizada dentro do Red Hat OpenShift, implantando um servidor Git interno (**Gitea**) compatível com restrições de segurança *rootless*, migrando repositórios de código-fonte e utilizando o paradigma nativo **Source-to-Image (S2I)** para compilar, versionar e publicar aplicações diretamente a partir do código, eliminando a necessidade de escrever ou gerenciar Dockerfiles.

**Cenário Corporativo (FinCorp Fábrica de Software Soberana):**  
Em instituições financeiras e ambientes de alta conformidade (*air-gapped*), equipes de desenvolvimento são proibidas de enviar código-fonte para provedores públicos de Git em nuvem (como GitHub ou GitLab) por razões de sigilo bancário e soberania de dados. A engenharia de plataformas da *FinCorp* determinou a implantação de uma infraestrutura de controle de versão privada hospedada dentro do próprio cluster OpenShift. O desafio é subir uma instância do **Gitea Rootless** que respeite as rígidas políticas de segurança do OpenShift (*Security Context Constraints - restricted-v2*), migrar a aplicação bancária legada e configurar o pipeline S2I para gerar imagens de contêiner prontas para produção a cada commit.

**Habilidades Conquistadas:**
1. Implantar aplicações de infraestrutura (*Gitea*) operando sob o padrão de segurança desprivilegiado (*rootless* na porta 3000).
2. Compreender a barreira de segurança imposta pelas *Security Context Constraints (SCC)* do OpenShift e por que imagens com usuário root são bloqueadas.
3. Criar rotas corporativas personalizadas com direcionamento explícito para portas de serviço desprivilegiadas (`--port=3000`).
4. Operar o painel administrativo do Gitea para provisionamento de contas e migração interna de repositórios Git.
5. Disparar a esteira de compilação automatizada **Source-to-Image (S2I)** a partir de um repositório Git interno via `oc new-app`.
6. Auditar o ciclo de vida do *Build Pod* efêmero, inspecionando logs de compilação do *BuildConfig* e catálogo de tags no *ImageStream*.
7. Executar testes de ciclo de vida contínuo, alterando código no Git e acompanhando o rollout automático sem downtime.

---

## 📋 Pré-requisitos & Materiais

* Acesso à estação de gerenciamento `workstation` do curso DO180 na Red Hat Academy.
* Cluster OpenShift ativo e autenticado via CLI:
  ```bash
  oc login -u developer -p developer https://api.ocp4.example.com:6443
  ```
* Acesso ao OpenShift Web Console no Firefox (`admin` / `redhatocp`):
  * `https://console-openshift-console.apps.ocp4.example.com`
* Repositório de referência para migração interna:
  * `https://github.com/sclorg/nodejs-ex.git`

---

## 🚀 Passo a Passo Guiado

### Passo 1: Autenticação e Preparação do Ambiente

Na estação de gerenciamento (`workstation`), realize a autenticação com o cluster OpenShift para estabelecer a sessão do Kubeconfig (`~/.kube/config`):

```bash
oc login -u developer -p developer https://api.ocp4.example.com:6443
```

> **Saída Esperada:**
> ```text
> Login successful.
> 
> You have access to the following projects and can switch between them with 'oc project <projectname>':
> 
> Using project "default".
> ```

Em seguida, selecione seu projeto dedicado e faça a limpeza de recursos anteriores para iniciar com o ambiente limpo:

```bash
oc project lab-open-shift-seunome || oc new-project lab-open-shift-seunome
oc delete all --all
oc delete pvc --all
```

> **Saída Esperada:**
> ```text
> Now using project "lab-open-shift-seunome" on server "https://api.ocp4.example.com:6443".
> pod "health-app-..." deleted
> service "health-app" deleted
> deployment.apps "health-app" deleted
> persistentvolumeclaim "health-app-claim" deleted
> ```

---

### Passo 2: Provisionamento do Servidor Git Privado (Gitea Rootless)

Por padrão de segurança, o Red Hat OpenShift impõe a SCC `restricted-v2`, que **proíbe terminantemente** contêineres de executarem com UID `0` (root) ou de realizarem bind em portas de rede privilegiadas (abaixo de 1024).

Utilizaremos a imagem oficial do **Gitea** na variante **rootless**, projetada especificamente para rodar sob UID numérico arbitrário e escutar nativamente na porta `3000`:

```bash
oc new-app gitea/gitea:latest-rootless --name=meu-git
```

> **Saída Esperada:**
> ```text
> --> Found container image ... (gitea/gitea:latest-rootless)
> --> Creating resources ...
>     deployment.apps "meu-git" created
>     service "meu-git" created
> --> Success
> ```

---

### Passo 3: Exposição de Rota Pública na Porta 3000

Como o serviço do Gitea roda na porta interna `3000`, devemos instruir o comando `oc expose` a direcionar o tráfego do HAProxy para a porta correta:

```bash
oc expose svc/meu-git --port=3000
```

> **Saída Esperada:**
> ```text
> route.route.openshift.io/meu-git exposed
> ```

Descubra a URL pública atribuída pelo roteador Ingress do OpenShift:

```bash
oc get route meu-git
```

> **Saída Esperada:**
> ```text
> NAME      HOST/PORT                                                         PATH   SERVICES   PORT   TERMINATION   WILDCARD
> meu-git   meu-git-lab-open-shift-seunome.apps.ocp4.example.com                    meu-git    3000                 None
> ```

Verifique se o Pod do Gitea atingiu o estado `Running` e `Ready 1/1`:

```bash
oc get pods -l deployment=meu-git
```

---

### Passo 4: Configuração Inicial do Gitea no Web Browser

1. No navegador **Firefox** da `workstation`, abra a URL da rota obtida no passo anterior:  
   `http://meu-git-lab-open-shift-seunome.apps.ocp4.example.com`
2. Você verá a página de instalação e configuração inicial do Gitea (*Initial Configuration*).
3. **Mantenha os padrões:**
   * **Database Type:** `SQLite3` (perfeito para execução em memória/lab sem dependência de banco externo).
   * **Application URL:** Manter a URL da rota do OpenShift preenchida automaticamente.
4. Role até o final da página e clique no botão azul: **"Install Gitea"**.
5. O Gitea inicializará as tabelas do banco em poucos segundos e redirecionará para a tela inicial.
6. Clique em **Register** (topo superior direito) para cadastrar sua conta:
   * **Username:** `aluno` (ou seu identificador preferido)
   * **Email:** `aluno@example.com`
   * **Password:** `P@ssw0rd123`
   * **Confirm Password:** `P@ssw0rd123`
7. Clique em **Register Account**. O primeiro usuário cadastrado torna-se automaticamente o Administrador da plataforma.

---

### Passo 5: Migração Ágil do Código-Fonte para o Git Privado

Agora que dispomos de um servidor Git corporativo dentro da malha do OpenShift, vamos trazer a aplicação bancária:

1. No painel do Gitea, clique no ícone **"+"** no menu superior direito e selecione **New Migration**.
2. Na tela de migração:
   * **Clone Address:** `https://github.com/sclorg/nodejs-ex.git`
   * **Repository Name:** `meu-app-nodejs`
   * Certifique-se de que a visibilidade seja **Public** (para permitir leitura pelo builder S2I sem necessidade de token SSH/chave privada).
3. Clique em **Migrate Repository**.
4. Em instantes, o repositório será clonado e exibirá os arquivos do projeto (`package.json`, `app.js`, `public/index.html`).
5. Copie a URL HTTP de clone do repositório exibida no topo da página:  
   Exemplo: `http://meu-git-lab-open-shift-seunome.apps.ocp4.example.com/aluno/meu-app-nodejs.git`

---

### Passo 6: Disparo da Fábrica de Software Source-to-Image (S2I)

O motor **Source-to-Image (S2I)** do OpenShift automatiza a criação de imagens combinando o código-fonte com uma imagem base homologada (*Builder Image*).

No terminal da `workstation`, crie a aplicação apontando diretamente para o seu repositório no Gitea (substitua a URL pela sua URL do Gitea obtida no passo anterior):

```bash
oc new-app http://meu-git-lab-open-shift-seunome.apps.ocp4.example.com/aluno/meu-app-nodejs.git --name=fabrica-app
```

> **Saída Esperada:**
> ```text
> --> Found image ... (nodejs:20-ubi9) in image stream "openshift/nodejs"
>     Node.js 20 
>     ---------- 
>     Platform for building and running Node.js 20 applications
> 
> --> Creating resources ...
>     imagestream.image.openshift.io "fabrica-app" created
>     buildconfig.openshift.io "fabrica-app" created
>     deployment.apps "fabrica-app" created
>     service "fabrica-app" created
> --> Success
>     Build scheduled, use 'oc logs -f buildconfig/fabrica-app' to track its progress.
> ```

*O que aconteceu nos bastidores:*  
1. O OpenShift analisou o código do repositório, detectou o arquivo `package.json` e selecionou automaticamente a imagem base oficial `nodejs` do catálogo interno.
2. Criou um **BuildConfig (`bc/fabrica-app`)**, que é a receita formal de compilação.
3. Criou um **ImageStream (`is/fabrica-app`)**, que gerenciará as versões da imagem compilada.
4. Disparou um **Build Pod efêmero** (`fabrica-app-1-build`) para clonar o código, baixar dependências `npm` e montar a imagem final.

---

### Passo 7: Telemetria e Auditoria de Compilação no Build Pod

Acompanhe a compilação do código em tempo real transmitindo os logs do BuildConfig:

```bash
oc logs -f bc/fabrica-app
```

> **Trechos Relevantes da Saída dos Logs:**
> ```text
> Cloning "http://meu-git-lab-open-shift-seunome.apps.ocp4.example.com/aluno/meu-app-nodejs.git" ...
> ---> Installing application source ...
> ---> Installing all dependencies ...
> Running 'npm install --production' ...
> added 50 packages in 4.2s
> ---> Cleaning up unused libraries ...
> Pushing image image-registry.openshift-image-registry.svc:5000/lab-open-shift-seunome/fabrica-app:latest ...
> Successfully pushed image-registry.openshift-image-registry.svc:5000/lab-open-shift-seunome/fabrica-app:latest
> Push successful
> ```

Assim que a imagem final é enviada para o registro interno, o OpenShift destrói o Pod efêmero de build e dispara automaticamente o rollout do Pod de produção.

Valide a transição para o Pod de execução:

```bash
oc get pods -l deployment=fabrica-app
```

> **Saída Esperada:**
> ```text
> NAME                           READY   STATUS      RESTARTS   AGE
> fabrica-app-1-build            0/1     Completed   0          95s
> fabrica-app-7b9c6f5d84-z4k2p   1/1     Running     0          20s
> ```

---

### Passo 8: Publicação e Teste de Ciclo de Vida Contínuo

1. Crie uma rota pública para a aplicação compilada pelo S2I:
   ```bash
   oc expose svc/fabrica-app
   oc get route fabrica-app
   ```
2. Abra a URL da rota `fabrica-app` no navegador Firefox. A página de exemplo do Node.js será exibida.
3. **Teste de Ciclo de Vida (Commit & Build):**
   * Retorne ao painel do **Gitea** no navegador.
   * Navegue no repositório `meu-app-nodejs` até o diretório `public/`.
   * Clique no arquivo `index.html` e depois no ícone de lápis para **Editar**.
   * Localize a tag `<h1>` (que contém originalmente `Node.js Crud Application`) e altere o título principal para:  
     `<h1>MBA MultiCloud FIAP - Fabrica S2I Soberana (Aluno: SeuNome)</h1>`
   * Role até o rodapé da página e clique em **Commit Changes**.
4. Dispare uma nova compilação na esteira S2I:
   ```bash
   oc start-build fabrica-app
   ```
5. Acompanhe a subida do build `fabrica-app-2` e o rollout automático:
   ```bash
   oc logs -f bc/fabrica-app
   oc rollout status deployment/fabrica-app
   ```
6. Recarregue a página da aplicação no navegador (pressione `F5` ou `Ctrl+F5`) e comprove a atualização instantânea em produção!

---

## 🧪 Validação & Critérios de Aceite

O laboratório é considerado concluído com sucesso quando:
* O servidor Git privado (`meu-git`) estiver ativo, acessível via rota pública na porta 3000 e hospedando o repositório `meu-app-nodejs`.
* O comando `oc get builds` comprovar a conclusão com sucesso (`Complete`) dos builds S2I.
* O comando `oc get pods -l deployment=fabrica-app` retornar o pod de produção em estado `Running` com `1/1` réplicas.
* A aplicação acessada pelo navegador exibir a mensagem customizada editada no Git pelo aluno.
* **Quick Win de Sala:** Compartilhar no chat da turma o print do navegador exibindo a aplicação no ar com o título customizado ou o print do painel do Gitea com o repositório migrado.

---

## 🧹 Cleanup & Próximos Passos

> [!NOTE]
> **Atenção:** **Não delete o servidor Gitea (`meu-git`)!**  
> A infraestrutura de Git privado será utilizada no **Lab 07 (Capstone Project)** para hospedar o código-fonte da aplicação integradora *Guestbook*.
> 
> Apenas desprovisione a aplicação temporária `fabrica-app` para liberar recursos de CPU/RAM no nó:
> ```bash
> oc delete all -l app=fabrica-app
> ```

---

## 💡 Desafios Complementares (Para Alunos Avançados)

1. **Anatomia Declarativa de BuildConfigs e Triggers de Automação:**  
   Extraia o manifesto declarativo do objeto BuildConfig:
   ```bash
   oc get bc fabrica-app -o yaml
   ```
   Localize a seção `spec.triggers` e identifique os dois mecanismos padrão de automação:
   * `ConfigChange`: Dispara um novo build sempre que o manifesto do BuildConfig for editado.
   * `ImageChange`: Dispara um novo build automaticamente se a imagem base de Node.js for atualizada pela Red Hat com patches de segurança (CVEs).

2. **Configuração de Webhook Nativo entre Gitea e OpenShift:**  
   Localize no manifesto do BuildConfig o segredo de webhook genérico (`Generic Webhook`):
   ```bash
   oc describe bc fabrica-app | grep -A 2 -i "Webhook Generic"
   ```
   Copie a URL de Webhook gerada e cadastre-a no painel do Gitea em **Settings -> Webhooks -> Add Webhook (Gitea)**. Realize uma nova alteração de texto no código e faça o commit para demonstrar o acionamento 100% automático (*push-button zero-click*) da esteira S2I sem precisar digitar `oc start-build` no terminal.
