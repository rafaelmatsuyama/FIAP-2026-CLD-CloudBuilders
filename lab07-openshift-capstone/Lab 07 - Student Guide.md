# Lab 07 - Integrated Multi-Tier Architecture on Red Hat OpenShift: Capstone Guestbook with MySQL and S2I

* **Program:** MBA in MultiCloud Strategy & Architecture
* **Environment / Platform:** Red Hat Academy (DO180 v4.14 / Red Hat OpenShift Container Platform 4.14)
* **Technical Stack:** OpenShift Multi-Tier, MySQL Stateful (`mysql:latest`), Kubernetes Secrets (`db-pass`), PVC (1Gi), Internal DNS Service Discovery, Source-to-Image (S2I), Ingress Routes, Health Probes, Chaos Engineering
* **Estimated Duration:** 45 to 60 minutes (Self-Guided Capstone Challenge)

---

## 🎯 Lab Objectives

Equip the engineer to integrate and consolidate all core architectural competencies mastered throughout the MBA program (declarative deployment orchestration, secret management, continuous Source-to-Image build pipelines, internal DNS service discovery, autonomous health probes, and durable block storage persistence) into an **Enterprise-Grade Capstone Project**: shipping a resilient multi-tier banking web application (*Guestbook*) consisting of an interactive Node.js frontend and a persistent MySQL backend immune to catastrophic node/pod failures.

**Enterprise Scenario (FinCorp Capstone Mission):**  
The *FinCorp* Enterprise Architecture Board has issued the final engineering mandate of the cloud modernization program: deploy the corporate audit and guest ledger platform (*Guestbook*) adhering to the highest cloud-native reliability standards. The architecture strictly enforces architectural separation of concerns:
1. **Data Layer (Stateful Backend):** An enterprise MySQL relational database backed by a durable 1Gi persistent volume claim (PVC) and secured with Kubernetes Secrets, isolated from public ingress and discoverable strictly within the cluster network fabric.
2. **Application Layer (Stateless Frontend):** A Node.js web application built through the native Source-to-Image (S2I) pipeline directly from the internal private Git server (Gitea), discovering the database via internal Kubernetes DNS (`guestbook-db`).
3. **Resilience & Chaos Engineering:** Fully configured Liveness and Readiness Probes, validated through deliberate chaos injection (pod termination), proving zero transactional data loss and automated cluster self-healing.

**Acquired Competencies:**
1. Architect and deploy enterprise *Multi-Tier* topologies (Stateless Web Frontend + Stateful Relational Backend) on Red Hat OpenShift.
2. Provision relational database workloads backed by declarative persistent storage (*1Gi PVC*).
3. Secure infrastructure credentials using Kubernetes *Secrets*, eliminating plaintext secrets from source repositories and manifests.
4. Audit internal cluster *Service Discovery* and name resolution via `CoreDNS`.
5. Connect applications built via *Source-to-Image (S2I)* to private backend services using standardized environment variables.
6. Configure and calibrate *Liveness* and *Readiness Probes* to accommodate database connection pool warm-up delays.
7. Execute *Chaos Engineering* experiments, auditing replica controller reconciliation and transaction durability following forced pod termination.

---

## 📋 Prerequisites & Materials

* Access to the `workstation` VM in the Red Hat Academy DO180 lab environment.
* Private Git server (**Gitea**) operational and accessible (deployed in **Lab 06**).
* Active OpenShift CLI session authenticated as `developer`:
  ```bash
  oc login -u developer -p developer https://api.ocp4.example.com:6443
  ```
* Source repository for the Guestbook application to migrate into Gitea:
  * `https://github.com/IBM/node-s2i-openshift.git`

---

## 🚀 Guided Step-by-Step Walkthrough

### Step 1: Authentication and Workspace Setup

On the management station (`workstation`), authenticate with the OpenShift cluster API to establish your session configuration (`~/.kube/config`):

```bash
oc login -u developer -p developer https://api.ocp4.example.com:6443
```

> **Expected Output:**
> ```text
> Login successful.
> Using project "lab-open-shift-seunome".
> ```

Verify that your dedicated project is active and that the Gitea Git server from Lab 06 is running:

```bash
oc project lab-open-shift-seunome
oc get pods -l deployment=meu-git
```

> **Expected Output:**
> ```text
> NAME                       READY   STATUS    RESTARTS   AGE
> meu-git-6c9f7b5d84-x8k1p   1/1     Running   0          25m
> ```

---

### Step 2: Database Secret Provisioning

In accordance with *FinCorp* information security standards, database credentials must never be passed in plaintext. Create a dedicated Kubernetes `Secret` to store database passwords:

```bash
oc create secret generic db-pass --from-literal=password=P@ssw0rd123
```

> **Expected Output:**
> ```text
> secret/db-pass created
> ```

---

### Step 3: Stateful Data Tier Deployment (MySQL with Persistence)

Deploy the relational database tier using the official Red Hat container image stream (`mysql:latest`), running under the unprivileged standard (SCC `restricted-v2`), and declaratively attach an enterprise 1Gi *PersistentVolumeClaim (PVC)* to persist the `/var/lib/mysql/data` directory:

1. Deploy the database service from the certified image stream:
   ```bash
   oc new-app mysql:latest \
     --name=guestbook-db \
     -e MYSQL_USER=guestbook \
     -e MYSQL_DATABASE=guestbook \
     -e MYSQL_PASSWORD=P@ssw0rd123
   ```

   > **Expected Output:**
   > ```text
   > --> Found image ... in image stream "openshift/mysql" under tag "latest" for "mysql:latest"
   >     MySQL 8.0 
   > --> Creating resources ...
   >     deployment.apps "guestbook-db" created
   >     service "guestbook-db" created
   > --> Success
   > ```

2. Attach an enterprise persistent volume (1Gi PVC) mounted at `/var/lib/mysql/data`:
   ```bash
   oc set volume deployment/guestbook-db \
     --add --name=db-storage \
     -t pvc --claim-size=1Gi \
     --mount-path=/var/lib/mysql/data
   ```

   > **Expected Output:**
   > ```text
   > deployment.apps/guestbook-db volume updated
   > ```

3. Monitor the storage allocation and database rollout completion:
   ```bash
   oc get pvc
   oc rollout status deployment/guestbook-db
   ```

   > **Expected Output:**
   > ```text
   > NAME               STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   AGE
   > guestbook-db-claim Bound    pvc-8c1b4e2a-5d9f-4123-b7c2-9e1a4d8f0b2c   1Gi        RWO            gp2            25s
   > deployment "guestbook-db" successfully rolled out
   > ```

The `guestbook-db` service is registered within internal cluster DNS listening on port `3306`. Notice that this service has no public Route, enforcing L4 network isolation within the project namespace.

---

### Step 4: Migrating Guestbook Codebase into Private Git (Gitea)

Now that the data tier is operational, migrate the web application repository into your private Gitea server:

1. In the **Firefox** browser on `workstation`, open your **Gitea** web console:  
   (Retrieve the route if needed via `oc get route meu-git`).
2. Log in with your corporate account (`aluno` / `P@ssw0rd123`).
3. Click the **"+"** icon in the top-right header and select **New Migration**.
4. Configure migration settings:
   * **Clone Address:** `https://github.com/IBM/node-s2i-openshift.git`
   * **Repository Name:** `guestbook-app`
   * **Visibility:** `Public`
5. Click **Migrate Repository**.
6. Copy the HTTP clone URL provided by Gitea:  
   Example: `http://meu-git-lab-open-shift-seunome.apps.ocp4.example.com/aluno/guestbook-app.git`

---

### Step 5: S2I Compilation and Multi-Tier Parameter Injection

Trigger the Source-to-Image (S2I) build pipeline pointing to your internal Gitea repository. Because the application source code and `package.json` reside within the `/site` subdirectory, supply the `--context-dir=site` flag and bind the build to the official `nodejs` builder image stream:

```bash
oc new-app nodejs~http://$(oc get route meu-git -o jsonpath='{.spec.host}')/aluno/guestbook-app.git \
  --context-dir=site \
  --name=guestbook-frontend
```

> **Expected Output:**
> ```text
> --> Found image ... in image stream "openshift/nodejs" under tag "latest" for "nodejs"
> --> Creating resources ...
>     imagestream.image.openshift.io "guestbook-frontend" created
>     buildconfig.openshift.io "guestbook-frontend" created
>     deployment.apps "guestbook-frontend" created
>     service "guestbook-frontend" created
> --> Success
> ```

Wire the web frontend to the database service by injecting configuration parameters and referencing the secret:

```bash
oc set env deployment/guestbook-frontend \
  DB_HOST=guestbook-db \
  DB_PORT=3306 \
  DB_USER=guestbook \
  DB_NAME=guestbook

oc set env deployment/guestbook-frontend --from=secret/db-pass
```

> **Expected Output:**
> ```text
> deployment.apps/guestbook-frontend updated
> ```

Stream build logs until compilation completes:

```bash
oc logs -f bc/guestbook-frontend
```

---

### Step 6: Health Probes Configuration and Public Ingress Exposure

1. Configure **Liveness** and **Readiness Probes** on the frontend deployment. Set a 10-second initial delay on the readiness probe to allow the Node.js application to establish its database connection pool:
   ```bash
   oc set probe deployment/guestbook-frontend --liveness --get-url=http://:8080/ --initial-delay-seconds=30
   oc set probe deployment/guestbook-frontend --readiness --get-url=http://:8080/ --initial-delay-seconds=10
   ```

2. Expose the frontend service to external traffic via the HAProxy Ingress router:
   ```bash
   oc expose svc/guestbook-frontend
   ```

3. Retrieve the generated public route FQDN:
   ```bash
   oc get route guestbook-frontend
   ```

   > **Expected Output:**
   > ```text
   > NAME                 HOST/PORT                                                               SERVICES             PORT       TERMINATION   WILDCARD
   > guestbook-frontend   guestbook-frontend-lab-open-shift-seunome.apps.ocp4.example.com         guestbook-frontend   8080-tcp                 None
   > ```

4. Verify that all components have reached healthy running states:
   ```bash
   oc get pods
   ```

   > **Expected Output:**
   > ```text
   > NAME                                  READY   STATUS      RESTARTS   AGE
   > guestbook-db-1-xxxxx                  1/1     Running     0          5m
   > guestbook-frontend-1-build            0/1     Completed   0          3m
   > guestbook-frontend-7c9d6f5b84-y4k1p   1/1     Running     0          1m
   > meu-git-6c9f7b5d84-x8k1p              1/1     Running     0          35m
   > ```

---

### Step 7: Functional Verification and Chaos Engineering Audit

1. In Firefox on `workstation`, open the `guestbook-frontend` route URL:  
   `http://guestbook-frontend-lab-open-shift-seunome.apps.ocp4.example.com`
2. The authentication view of the corporate **Example Health** portal (`/login.html`) will load, connected to the backend database tier.
3. To access the corporate portal, input the official credentials into the fields:
   * **Name:** `aluno@example.com`
   * **Password:** `aluno`
4. Click the **Sign In** button. The medical records and patient analytics portal will load, consuming data from the relational database tier.
5. **The Chaos Engineering Experiment (State Durability Audit):**  
   Simulate a catastrophic database failure by forcibly terminating the running database pod:
   ```bash
   oc delete pod -l deployment=guestbook-db
   ```
6. In the terminal, monitor the OpenShift controller immediately scheduling a replacement pod:
   ```bash
   oc get pods -l deployment=guestbook-db -w
   ```
   *The terminated pod transitions through `Terminating` while a new replica is immediately launched (`ContainerCreating` $\rightarrow$ `Running 1/1`).*
7. **Validating Data Integrity:**  
   Return to Firefox and refresh the application page (`F5`).  
   **Result:** The application remains operational and fully responsive! Because state is stored on the bound *PersistentVolumeClaim (PVC)*, the replacement pod automatically remounted the durable filesystem, proving **zero transactional data loss and total immunity to pod ephemerality**.

---

## 🧪 Validation & Acceptance Criteria

The laboratory is successfully completed when:
* The database tier (`guestbook-db`) is active with a 1Gi PVC in `Bound` status.
* The frontend (`guestbook-frontend`) is compiled via S2I and communicating with the database through internal DNS and secrets.
* The public route loads the application and successfully authenticates into the Example Health portal with `aluno@example.com` / `aluno`.
* The chaos test proves that the application remains online and transactional data persists after the database pod is terminated.
* **Classroom Quick Win:** Share a screenshot of the browser displaying the authenticated Example Health portal alongside the Web Console *Topology View* showing the interconnected multi-tier architecture.

---

## 🧹 Cleanup & Course Wrap-Up

Upon finishing the capstone evaluation, clean up the project namespace:

```bash
oc delete all --all
oc delete pvc --all
oc delete secret db-pass
```

> **Expected Output:**
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

## 💡 Complementary Challenges (For Advanced Students)

1. **Internal DNS Name Resolution Auditing (`CoreDNS`):**  
   Open an interactive shell inside the running frontend pod:
   ```bash
   oc rsh deployment/guestbook-frontend
   ```
   Inspect DNS resolution parameters inside the container:
   ```bash
   getent hosts guestbook-db
   cat /etc/resolv.conf
   exit
   ```
   Demonstrate that `guestbook-db` maps directly to the virtual *ClusterIP* of the internal service, and explain how the search domain `search lab-open-shift-seunome.svc.cluster.local` decouples pods from static IP configurations.

2. **Horizontal Elastic Scaling on the Stateless Web Tier:**  
   Scale the web frontend to 3 balanced replicas:
   ```bash
   oc scale deployment/guestbook-frontend --replicas=3
   oc get pods -l deployment=guestbook-frontend
   ```
   Submit new messages through the browser. Explain why the web tier can scale horizontally with zero session conflict, grounding the **Stateless Frontend + Stateful Backend** architectural paradigm.
