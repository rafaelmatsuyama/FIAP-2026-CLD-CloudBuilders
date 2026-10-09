# Lab 07 - Arquitetura Multi-Tier Integrada no Red Hat OpenShift: Missão Final Guestbook com MySQL e S2I

* **Programa:** MBA em MultiCloud Strategy & Architecture
* **Ambiente / Plataforma:** Red Hat Academy (DO180 v4.14 / Red Hat OpenShift Container Platform 4.14)
* **Stack Técnica:** OpenShift Multi-Tier, MySQL Stateful (`mysql:latest`), Kubernetes Secrets (`db-pass`), PVC (1Gi), Internal DNS Service Discovery, Source-to-Image (S2I), Ingress Routes, Health Probes, Chaos Engineering
* **Duração Estimada:** 45 a 60 minutos (Desafio Integrador Capstone Autoguiado)

---

## 🎯 Objetivo do Lab

Capacitar o aluno a integrar e consolidar todos os pilares arquiteturais dominados ao longo do MBA (orquestração declarativa, desacoplamento de segredos, fábrica de software contínua S2I, descoberta de serviços via DNS interno, sondas de saúde e persistência de dados) em um projeto prático de **Nível Enterprise**: a entrega de uma aplicação bancária multi-tier (*Guestbook*) composta por um frontend web interativo e um banco de dados relacional resiliente a falhas catastróficas.

**Cenário Corporativo (FinCorp Missão Final Capstone):**  
O comitê executivo de arquitetura da *FinCorp* convocou sua equipe para a prova de fogo da jornada de modernização de plataformas: entregar o sistema bancário corporativo de auditoria e registros (*Guestbook*) operando sob os mais rigorosos padrões de confiabilidade *cloud-native*. A solução exige a separação estrita de responsabilidades:
1. **Camada de Dados (Stateful):** Um banco de dados MySQL corporativo com persistência real em disco (PVC de 1Gi) e credenciais blindadas via Secret, isolado na rede interna e sem exposição pública direta.
2. **Camada de Aplicação (Stateless):** Um frontend Node.js compilado pela esteira Source-to-Image (S2I) a partir do servidor Git privado (Gitea) do cluster, conectando-se ao banco via resolução de nomes do DNS interno do Kubernetes (`guestbook-db`).
3. **Resiliência & Engenharia de Caos:** Probes de Liveness/Readiness configuradas e validação obrigatória através de um teste de falha deliberada (destruição do pod do banco de dados), comprovando tolerância a desastres e zero perda transacional.

**Habilidades Conquistadas:**
1. Arquitetar e implantar topologias corporativas *Multi-Tier* (Frontend Web + Backend Relacional) no Red Hat OpenShift.
2. Provisionar bancos de dados relacionais corporativos com volumes persistentes declarativos (*PVC de 1Gi*).
3. Blindar credenciais de infraestrutura utilizando Kubernetes *Secrets*, eliminando senhas em texto puro de manifests e código.
4. Explorar e auditar o mecanismo nativo de *Service Discovery* e DNS interno do OpenShift (`CoreDNS`).
5. Conectar aplicações compiladas via *Source-to-Image (S2I)* a serviços internos por meio de variáveis de ambiente padronizadas.
6. Calibrar sondas de *Liveness* e *Readiness Probes* para sincronização de ciclo de vida e warm-up de conexões de banco de dados.
7. Executar experimentos de *Engenharia de Caos*, auditando a reconciliação automática do ReplicaSet e a persistência de transações após a destruição forçada de pods de banco de dados.

---

## 📋 Pré-requisitos & Materiais

* Acesso à estação de gerenciamento `workstation` do curso DO180 na Red Hat Academy.
* Servidor Git corporativo (**Gitea**) ativo e acessível (provisionado no **Lab 06**).
* Sessão CLI ativa e autenticada como `developer`:
  ```bash
  oc login -u developer -p developer https://api.ocp4.example.com:6443
  ```
* Repositório de código-fonte da aplicação Guestbook para migração no Gitea:
  * `https://github.com/IBM/node-s2i-openshift.git`

---

## 🚀 Passo a Passo Guiado

### Passo 1: Autenticação e Preparação do Workspace

Na estação de gerenciamento (`workstation`), realize o login no cluster OpenShift para restabelecer o arquivo de configuração local (`~/.kube/config`):

```bash
oc login -u developer -p developer https://api.ocp4.example.com:6443
```

> **Saída Esperada:**
> ```text
> Login successful.
> Using project "lab-open-shift-seunome".
> ```

Certifique-se de que o seu projeto dedicado esteja selecionado e verifique se o servidor Gitea (`meu-git`) do Lab 06 está ativo:

```bash
oc project lab-open-shift-seunome
oc get pods -l deployment=meu-git
```

> **Saída Esperada:**
> ```text
> NAME                       READY   STATUS    RESTARTS   AGE
> meu-git-6c9f7b5d84-x8k1p   1/1     Running   0          25m
> ```

---

### Passo 2: Criação de Segredos Corporativos da Base de Dados

Em conformidade com a política de segurança bancária da *FinCorp*, credenciais de acesso ao banco nunca devem trafegar em texto claro. Crie um objeto `Secret` dedicado para armazenar a senha administrativa e de conexão do banco de dados:

```bash
oc create secret generic db-pass --from-literal=password=P@ssw0rd123
```

> **Saída Esperada:**
> ```text
> secret/db-pass created
> ```

---

### Passo 3: Provisionamento da Camada de Dados (MySQL com Persistência)

Provisionaremos a instância relacional utilizando o ImageStream oficial da Red Hat (`mysql:latest`), que opera sob o padrão de segurança não-privilegiado (SCC `restricted-v2`), e em seguida anexaremos declarativamente um *PersistentVolumeClaim (PVC)* corporativo de 1Gi para persistir o diretório `/var/lib/mysql/data`:

1. Instancie o banco de dados a partir do ImageStream corporativo:
   ```bash
   oc new-app mysql:latest \
     --name=guestbook-db \
     -e MYSQL_USER=guestbook \
     -e MYSQL_DATABASE=guestbook \
     -e MYSQL_PASSWORD=P@ssw0rd123
   ```

   > **Saída Esperada:**
   > ```text
   > --> Found image ... in image stream "openshift/mysql" under tag "latest" for "mysql:latest"
   >     MySQL 8.0 
   > --> Creating resources ...
   >     deployment.apps "guestbook-db" created
   >     service "guestbook-db" created
   > --> Success
   > ```

2. Anexe o volume persistente corporativo (PVC de 1Gi) montado em `/var/lib/mysql/data`:
   ```bash
   oc set volume deployment/guestbook-db \
     --add --name=db-storage \
     -t pvc --claim-size=1Gi \
     --mount-path=/var/lib/mysql/data
   ```

   > **Saída Esperada:**
   > ```text
   > deployment.apps/guestbook-db volume updated
   > ```

3. Acompanhe a alocação do volume persistente e a conclusão do rollout do banco:
   ```bash
   oc get pvc
   oc rollout status deployment/guestbook-db
   ```

   > **Saída Esperada:**
   > ```text
   > NAME               STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   AGE
   > guestbook-db-claim Bound    pvc-8c1b4e2a-5d9f-4123-b7c2-9e1a4d8f0b2c   1Gi        RWO            gp2            25s
   > deployment "guestbook-db" successfully rolled out
   > ```

O serviço interno `guestbook-db` foi registrado no DNS do cluster escutando na porta padrão (`3306`). Observe que este banco **não possui rota pública**, garantindo isolamento L4/L7 dentro do namespace.

---

### Passo 4: Migração do Código do Guestbook no Git Corporativo (Gitea)

Agora que a camada de dados está pronta, vamos trazer o código-fonte da aplicação web para o nosso servidor Git privado:

1. No navegador **Firefox** da `workstation`, abra o painel do seu **Gitea**:  
   (Utilize a rota do Gitea obtida no Lab 06 via `oc get route meu-git`).
2. Faça login com sua conta corporativa cadastrada (`aluno` / `P@ssw0rd123`).
3. Clique no ícone **"+"** no menu superior direito e selecione **New Migration**.
4. Preencha os dados de importação:
   * **Clone Address:** `https://github.com/IBM/node-s2i-openshift.git`
   * **Repository Name:** `guestbook-app`
   * **Visibility:** `Public`
5. Clique em **Migrate Repository**.
6. Copie a URL HTTP de clone do repositório gerada pelo Gitea:  
   Exemplo: `http://meu-git-lab-open-shift-seunome.apps.ocp4.example.com/aluno/guestbook-app.git`

---

### Passo 5: Compilação S2I e Injeção de Parâmetros Multi-Tier

Dispare a esteira Source-to-Image (S2I) apontando para o seu repositório no Gitea. Como o código-fonte da aplicação e o arquivo `package.json` estão localizados no subdiretório `/site` do repositório, utilizamos a flag `--context-dir=site` e vinculamos o builder ao ImageStream oficial `nodejs`:

```bash
oc new-app nodejs~http://$(oc get route meu-git -o jsonpath='{.spec.host}')/aluno/guestbook-app.git \
  --context-dir=site \
  --name=guestbook-frontend
```

> **Saída Esperada:**
> ```text
> --> Found image ... in image stream "openshift/nodejs" under tag "latest" for "nodejs"
> --> Creating resources ...
>     imagestream.image.openshift.io "guestbook-frontend" created
>     buildconfig.openshift.io "guestbook-frontend" created
>     deployment.apps "guestbook-frontend" created
>     service "guestbook-frontend" created
> --> Success
> ```

Agora configure a conectividade do frontend com o banco injetando as variáveis de ambiente necessárias e a senha a partir do Secret:

```bash
oc set env deployment/guestbook-frontend \
  DB_HOST=guestbook-db \
  DB_PORT=3306 \
  DB_USER=guestbook \
  DB_NAME=guestbook

oc set env deployment/guestbook-frontend --from=secret/db-pass
```

> **Saída Esperada:**
> ```text
> deployment.apps/guestbook-frontend updated
> ```

Acompanhe o log da compilação da imagem da aplicação:

```bash
oc logs -f bc/guestbook-frontend
```

Aguarde a mensagem `Push successful` e a conclusão do build pod.

---

### Passo 6: Configuração de Sondas de Saúde e Exposição de Ingress

1. Configure as sondas de **Liveness** e **Readiness** no frontend. Observe o delay de 10s no Readiness para dar tempo da aplicação estabelecer o pool de conexões com o MySQL:
   ```bash
   oc set probe deployment/guestbook-frontend --liveness --get-url=http://:8080/ --initial-delay-seconds=30
   oc set probe deployment/guestbook-frontend --readiness --get-url=http://:8080/ --initial-delay-seconds=10
   ```

2. Exponha o serviço do frontend para acesso externo através do roteador Ingress (HAProxy):
   ```bash
   oc expose svc/guestbook-frontend
   ```

3. Obtenha a URL pública gerada:
   ```bash
   oc get route guestbook-frontend
   ```

   > **Saída Esperada:**
   > ```text
   > NAME                 HOST/PORT                                                               SERVICES             PORT       TERMINATION   WILDCARD
   > guestbook-frontend   guestbook-frontend-lab-open-shift-seunome.apps.ocp4.example.com         guestbook-frontend   8080-tcp                 None
   > ```

4. Verifique a estabilização de todos os pods da arquitetura:
   ```bash
   oc get pods
   ```

   > **Saída Esperada:**
   > ```text
   > NAME                                  READY   STATUS      RESTARTS   AGE
   > guestbook-db-7d8b9c6f5d-7km9q         1/1     Running     0          5m
   > guestbook-frontend-1-build            0/1     Completed   0          3m
   > guestbook-frontend-7c9d6f5b84-y4k1p   1/1     Running     0          1m
   > meu-git-6c9f7b5d84-x8k1p              1/1     Running     0          35m
   > ```

---

### Passo 7: Validação Funcional e Teste de Engenharia de Caos

1. No navegador Firefox da `workstation`, abra a URL da rota `guestbook-frontend`:  
   `http://guestbook-frontend-lab-open-shift-seunome.apps.ocp4.example.com`
2. A tela de autenticação do portal corporativo **Example Health** (`/login.html`) será exibida conectada à camada de dados.
3. Para acessar o painel corporativo, insira as credenciais oficiais nos campos:
   * **Name:** `aluno@example.com`
   * **Password:** `aluno`
4. Clique no botão **Sign In**. O painel de registros e prontuários médicos da aplicação carregará consumindo os dados da base relacional.
5. **O Teste de Caos (Resiliência & Persistência de Dados):**  
   Simule uma falha catastrófica no nó de banco de dados, destruindo o pod do banco em execução:
   ```bash
   oc delete pod -l deployment=guestbook-db
   ```
6. No terminal, observe imediatamente o controlador do OpenShift recriando um novo Pod de banco de dados:
   ```bash
   oc get pods -l deployment=guestbook-db -w
   ```
   *O pod destruído é finalizado (`Terminating`) e um novo pod é agendado imediatamente (`ContainerCreating` $\rightarrow$ `Running 1/1`).*
7. **Homologação da Integridade de Dados:**  
   Retorne ao navegador e recarregue a página da aplicação (`F5`).  
   **Resultado:** O painel continua acessível e operacional! Graças ao *PersistentVolumeClaim (PVC)*, o novo contêiner remontou o volume corporativo exatamente no estado anterior, comprovando **zero perda de dados e total imunidade à efemeridade de contêineres**.

---

## 🧪 Validação & Critérios de Aceite

O laboratório é considerado concluído com sucesso quando:
* O banco de dados (`guestbook-db`) estiver ativo com PVC de 1Gi em estado `Bound`.
* O frontend (`guestbook-frontend`) estiver compilado via S2I e conectado com sucesso ao banco através de variáveis e Secret.
* A rota pública do frontend abrir no navegador permitindo a autenticação no portal Example Health com as credenciais `aluno@example.com` / `aluno`.
* O teste de destruição do pod do banco demonstrar que a aplicação permanece operacional e os dados intactos após o reinício automático do Pod.
* **Quick Win de Sala / Entregável Final:** Capturar o print do navegador exibindo o portal Example Health autenticado e a visualização da *Topology View* no Web Console mostrando os nós interconectados.

---

## 🧹 Cleanup & Encerramento do Módulo

Após a conclusão e validação do laboratório integrador, execute a limpeza completa do namespace:

```bash
oc delete all --all
oc delete pvc --all
oc delete secret db-pass
```

> **Saída Esperada:**
> ```text
> pod "guestbook-db-..." deleted
> pod "guestbook-frontend-..." deleted
> service "guestbook-db" deleted
> service "guestbook-frontend" deleted
> route.route.openshift.io "guestbook-frontend" deleted
> persistentvolumeclaim "guestbook-db" deleted
> secret "db-pass" deleted
> ```

---

## 💡 Desafios Complementares (Para Alunos Avançados)

1. **Auditoria de Resolução de DNS Interno (`CoreDNS`):**  
   Abra uma sessão interativa dentro do contêiner do frontend web em execução:
   ```bash
   oc rsh deployment/guestbook-frontend
   ```
   Dentro do contêiner, consulte a resolução de nomes da camada de dados utilizando as ferramentas nativas de rede:
   ```bash
   getent hosts guestbook-db
   cat /etc/resolv.conf
   exit
   ```
   Comprove que o nome `guestbook-db` resolve diretamente para o endereço virtual (*ClusterIP*) do Service interno, e explique como a diretriz de busca de domínio `search lab-open-shift-seunome.svc.cluster.local` permite a comunicação entre pods sem acoplamento de IPs fixos.

2. **Escalonamento Elástico da Camada Web sem Conflito de Sessão:**  
   Escalone a camada de frontend para 3 réplicas balanceadas:
   ```bash
   oc scale deployment/guestbook-frontend --replicas=3
   oc get pods -l deployment=guestbook-frontend
   ```
   Acesse a aplicação no navegador e envie novas mensagens. Explique por que a camada web pôde ser escalada horizontalmente de forma instantânea sem degradação ou perda de sincronização, fundamentando o padrão arquitetural **Stateless Frontend + Stateful Backend**.
