# Lab 06 - Cloud-Native Software Factory and Private Git on Red Hat OpenShift: Rootless Gitea and S2I Pipelines

* **Program:** MBA in MultiCloud Strategy & Architecture
* **Environment / Platform:** Red Hat Academy (DO180 v4.14 / Red Hat OpenShift Container Platform 4.14)
* **Technical Stack:** Red Hat OpenShift S2I, BuildConfigs, ImageStreams, Gitea Rootless (port 3000), Security Context Constraints (SCC `restricted-v2`), OpenShift Routes, Node.js Builder
* **Estimated Duration:** 35 to 40 minutes

---

## 🎯 Lab Objectives

Equip the engineer to establish a sovereign, fully automated enterprise software factory inside Red Hat OpenShift by deploying an internal *rootless* Git server (**Gitea**), migrating enterprise source code repositories, and leveraging the native **Source-to-Image (S2I)** engine to build, version, and deploy container images directly from code without authoring or maintaining Dockerfiles.

**Enterprise Scenario (FinCorp Sovereign Software Factory):**  
In financial institutions and regulated *air-gapped* environments, engineering teams are strictly prohibited from pushing intellectual property and proprietary source code to public SaaS Git platforms (such as public GitHub or GitLab) due to regulatory data sovereignty and banking secrecy laws. The *FinCorp* Platform Engineering squad has mandated the deployment of a private version-control infrastructure hosted entirely within the OpenShift cluster. The mission is to deploy an enterprise-ready **Rootless Gitea** instance conforming to OpenShift's hardened security policies (*Security Context Constraints - restricted-v2* on non-privileged port 3000), migrate a core banking microservice, and wire an autonomous S2I pipeline that compiles, tags, and deploys production containers on every git commit.

**Acquired Competencies:**
1. Deploy platform infrastructure services (*Gitea*) complying with unprivileged container execution standards (*rootless* on port 3000).
2. Understand the security boundaries established by OpenShift's *Security Context Constraints (SCC)* and why containers requiring root privileges are blocked by default.
3. Create custom Ingress *Routes* bound explicitly to non-privileged service target ports (`--port=3000`).
4. Operate the Gitea administrative console for account provisioning and source repository migration.
5. Trigger automated **Source-to-Image (S2I)** build pipelines against internal cluster Git endpoints via `oc new-app`.
6. Audit ephemeral *Build Pod* lifecycles, monitoring compilation logs via `bc/fabrica-app` and tracking image versions via `is/fabrica-app`.
7. Execute continuous software delivery lifecycles, committing code changes in Git and observing zero-downtime rolling deployments.

---

## 📋 Prerequisites & Materials

* Access to the `workstation` VM in the Red Hat Academy DO180 lab environment.
* Active OpenShift CLI session authenticated as `developer`:
  ```bash
  oc login -u developer -p developer https://api.ocp4.example.com:6443
  ```
* OpenShift Web Console access via Firefox (`admin` / `redhatocp`):
  * `https://console-openshift-console.apps.ocp4.example.com`
* Upstream reference repository for migration:
  * `https://github.com/sclorg/nodejs-ex.git`

---

## 🚀 Guided Step-by-Step Walkthrough

### Step 1: Authentication and Workspace Setup

On the management station (`workstation`), authenticate with the OpenShift cluster API to establish your local session configuration (`~/.kube/config`):

```bash
oc login -u developer -p developer https://api.ocp4.example.com:6443
```

> **Expected Output:**
> ```text
> Login successful.
> 
> You have access to the following projects and can switch between them with 'oc project <projectname>':
> 
> Using project "default".
> ```

Next, ensure your dedicated project is active and cleared of previous workloads (replace `seunome` with your unique identifier):

```bash
oc project lab-open-shift-seunome || oc new-project lab-open-shift-seunome
oc delete all --all
oc delete pvc --all
```

> **Expected Output:**
> ```text
> Now using project "lab-open-shift-seunome" on server "https://api.ocp4.example.com:6443".
> pod "health-app-..." deleted
> service "health-app" deleted
> deployment.apps "health-app" deleted
> persistentvolumeclaim "health-app-claim" deleted
> ```

---

### Step 2: Deploying Private Rootless Git Infrastructure (Gitea)

By security design, Red Hat OpenShift enforces the `restricted-v2` SCC, which **strictly prohibits** containers from executing as UID `0` (root) or binding to privileged network ports (ports below 1024).

Deploy the official **Gitea** container using the **rootless** variant, designed specifically to operate under an arbitrary unprivileged UID and listen natively on port `3000`:

```bash
oc new-app gitea/gitea:latest-rootless --name=meu-git
```

> **Expected Output:**
> ```text
> --> Found container image ... (gitea/gitea:latest-rootless)
> --> Creating resources ...
>     deployment.apps "meu-git" created
>     service "meu-git" created
> --> Success
> ```

---

### Step 3: Exposing Public Ingress Route on Port 3000

Because Gitea runs its HTTP service on unprivileged internal port `3000`, configure `oc expose` to route external HAProxy Ingress traffic to the proper target port:

```bash
oc expose svc/meu-git --port=3000
```

> **Expected Output:**
> ```text
> route.route.openshift.io/meu-git exposed
> ```

Retrieve the public FQDN allocated by the OpenShift router:

```bash
oc get route meu-git
```

> **Expected Output:**
> ```text
> NAME      HOST/PORT                                                         PATH   SERVICES   PORT   TERMINATION   WILDCARD
> meu-git   meu-git-lab-open-shift-seunome.apps.ocp4.example.com                    meu-git    3000                 None
> ```

Verify that the Gitea pod has reached `Running` status with `1/1` containers ready:

```bash
oc get pods -l deployment=meu-git
```

---

### Step 4: Gitea Web Console Initial Configuration

1. In the **Firefox** browser on `workstation`, navigate to the Gitea URL obtained in the previous step:  
   `http://meu-git-lab-open-shift-seunome.apps.ocp4.example.com`
2. The Gitea **Initial Configuration** wizard will load.
3. **Keep all default parameters:**
   * **Database Type:** `SQLite3` (optimal for lab and ephemeral execution without external database dependencies).
   * **Application URL:** Keep the automatically populated OpenShift route URL.
4. Scroll to the bottom of the page and click **"Install Gitea"**.
5. Gitea will initialize its schema in seconds and redirect to the landing page.
6. Click **Register** (top-right corner) to create your user account:
   * **Username:** `aluno` (or your preferred username)
   * **Email:** `aluno@example.com`
   * **Password:** `P@ssw0rd123`
   * **Confirm Password:** `P@ssw0rd123`
7. Click **Register Account**. The first registered user automatically becomes the platform Administrator.

---

### Step 5: Migrating Application Code to Private Git

Now that sovereign Git infrastructure is operational inside the cluster network fabric, migrate the banking application:

1. In the Gitea top-right navigation menu, click the **"+"** icon and select **New Migration**.
2. Configure migration details:
   * **Clone Address:** `https://github.com/sclorg/nodejs-ex.git`
   * **Repository Name:** `meu-app-nodejs`
   * Ensure **Visibility** is set to **Public** (allowing unauthenticated read access from internal OpenShift build pods).
3. Click **Migrate Repository**.
4. Within moments, the repository will be fully cloned with all files (`package.json`, `app.js`, `public/index.html`).
5. Copy the HTTP clone URL displayed at the top of the repository page:  
   Example: `http://meu-git-lab-open-shift-seunome.apps.ocp4.example.com/aluno/meu-app-nodejs.git`

---

### Step 6: Triggering the Source-to-Image (S2I) Software Factory

OpenShift's native **Source-to-Image (S2I)** engine eliminates Dockerfile maintenance by pairing your raw application source code with an officially hardened Red Hat *Builder Image*.

In the terminal on `workstation`, deploy the application pointing directly to your internal Gitea repository (substitute with your actual Gitea clone URL):

```bash
oc new-app http://meu-git-lab-open-shift-seunome.apps.ocp4.example.com/aluno/meu-app-nodejs.git --name=fabrica-app
```

> **Expected Output:**
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

*Under the Hood Operations:*  
1. OpenShift inspected the repository, identified `package.json`, and selected the official `nodejs` builder image from the cluster catalog.
2. Created a **BuildConfig (`bc/fabrica-app`)** storing the formal build blueprint.
3. Created an **ImageStream (`is/fabrica-app`)** tracking image digests and versions.
4. Scheduled an ephemeral **Build Pod** (`fabrica-app-1-build`) to assemble dependencies and push the resulting container image into the integrated OpenShift Container Registry.

---

### Step 7: Build Pod Telemetry and Compilation Auditing

Stream real-time build execution logs directly from the BuildConfig:

```bash
oc logs -f bc/fabrica-app
```

> **Sample Compilation Log Output:**
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

Once image assembly completes and the container is pushed to the internal registry, OpenShift automatically destroys the ephemeral build pod and schedules the production deployment pod.

Verify the transition to the runtime container:

```bash
oc get pods -l deployment=fabrica-app
```

> **Expected Output:**
> ```text
> NAME                           READY   STATUS      RESTARTS   AGE
> fabrica-app-1-build            0/1     Completed   0          95s
> fabrica-app-7b9c6f5d84-z4k2p   1/1     Running     0          20s
> ```

---

### Step 8: Public Ingress and Continuous Delivery Validation

1. Expose the S2I-generated service with an external route:
   ```bash
   oc expose svc/fabrica-app
   oc get route fabrica-app
   ```
2. Open the `fabrica-app` route URL in Firefox to view the running Node.js web application.
3. **Continuous Lifecycle Test (Commit & Automated Rollout):**
   * Return to the **Gitea** web console in Firefox.
   * Inside the `meu-app-nodejs` repository, navigate to the `public/` directory.
   * Click on the `index.html` file and click the pencil icon to **Edit** the file.
   * Locate the `<h1>` tag (originally containing `Node.js Crud Application`) and update the primary header line to:  
     `<h1>MBA MultiCloud FIAP - Sovereign S2I Factory (Engineer: YourName)</h1>`
   * Scroll down and click **Commit Changes**.
4. Trigger a new compilation run in the S2I pipeline:
   ```bash
   oc start-build fabrica-app
   ```
5. Observe build `fabrica-app-2` stream logs and orchestrate a zero-downtime rolling update:
   ```bash
   oc logs -f bc/fabrica-app
   oc rollout status deployment/fabrica-app
   ```
6. Refresh the application page in Firefox (`F5` or `Ctrl+F5`) to observe the live update in production!

---

## 🧪 Validation & Acceptance Criteria

The laboratory is successfully completed when:
* The internal Git server (`meu-git`) is accessible via its route on port 3000 and hosts the migrated repository.
* `oc get builds` confirms that S2I builds have completed with status `Complete`.
* `oc get pods -l deployment=fabrica-app` verifies the production pod is `Running` with `1/1` containers ready.
* The web application displays the customized headline committed directly to Gitea.
* **Classroom Quick Win:** Share a screenshot of the browser displaying the customized S2I application or the Gitea console showing the migrated repository in the classroom chat.

---

## 🧹 Cleanup & Next Steps

> [!NOTE]
> **Important:** **Do not delete the Gitea server (`meu-git`)!**  
> The private Git infrastructure will be reused in **Lab 07 (Capstone Project)** to host the multi-tier *Guestbook* codebase.
> 
> Simply delete the temporary `fabrica-app` workload to reclaim CPU and memory:
> ```bash
> oc delete all -l app=fabrica-app
> ```

---

## 💡 Complementary Challenges (For Advanced Students)

1. **Declarative BuildConfig Anatomy and Automation Triggers:**  
   Extract the declarative YAML specification of the BuildConfig:
   ```bash
   oc get bc fabrica-app -o yaml
   ```
   Inspect the `spec.triggers` block to identify the standard build trigger mechanisms:
   * `ConfigChange`: Automatically initiates a new build whenever the BuildConfig specification is modified.
   * `ImageChange`: Automatically initiates a rebuild whenever the upstream Red Hat Node.js builder image receives security patches (CVE remediations).

2. **Native Webhook Integration Between Gitea and OpenShift:**  
   Locate the generic webhook URL from the BuildConfig:
   ```bash
   oc describe bc fabrica-app | grep -A 2 -i "Webhook Generic"
   ```
   Copy the webhook endpoint and register it inside Gitea under **Settings -> Webhooks -> Add Webhook (Gitea)**. Make another code modification and commit to verify that Gitea triggers a zero-click automatic build in OpenShift without issuing `oc start-build` in the terminal.
