# tekton-lab

POC di una pipeline CI/CD completa su **Red Hat OpenShift Pipelines**.
Migrata e validata su **OpenShift Pipelines Operator 1.23** (Tekton Pipelines 1.12.x,
Tekton Triggers 0.36.x) su OpenShift Container Platform 4.14+.

Pipeline: `clean-volume → clone-repository → maven-deploy → buildah (build+push) → deploy-app`,
triggerabile a mano, via `tkn`, via `PipelineRun` o via **EventListener** (GitHub webhook push).

---

## Compatibilità (OSP 1.23)

| Componente | Versione |
|---|---|
| Operator | 1.23 (channel `pipelines-1.23`, CSV `v1.23.x`) |
| Tekton Pipelines | 1.12.x — API `tekton.dev/v1` |
| Tekton Triggers | 0.36.x — API `triggers.tekton.dev/v1beta1` |
| OCP supportato | 4.14, 4.16, 4.18 – 4.22 |

> Le risorse del repo sono già alla versione API corretta: Task/Pipeline/PipelineRun in
> `tekton.dev/v1`, Trigger in `triggers.tekton.dev/v1beta1`. Non servono conversioni.

## 1. Installare l'operatore

__https://docs.redhat.com/en/documentation/red_hat_openshift_pipelines/1.23/html/installing_openshift_pipelines/index__

```sh
$ curl -s https://raw.githubusercontent.com/gmeghnag/tekton-lab/main/pipelines-operator/subscription.yaml | oc apply -f -
```

Il `Subscription` punta al channel **`pipelines-1.23`** (il catalogo Red Hat non espone un
channel `stable`, solo `latest` + `pipelines-1.X`). Se l'installplan risulta in `Manual`,
approvarlo:

```sh
$ oc get installplan -n openshift-operators          # trova quello del pipelines operator
$ oc patch installplan -n openshift-operators <installplan-name> --type merge -p '{"spec":{"approved":true}}'
$ oc get csv -n openshift-pipelines -w               # attendi phase=Succeeded
```

> Nota: l'operatore crea da solo il ServiceAccount `pipeline` nei namespace con le pipeline.

## 2. Installare il CLI `tkn`

Il mirror pubblico ha dismesso i path storici (`.../pipeline/0.17.2/...`). Usa il `latest`
del mirror o il client OCP (che include `tkn`):

```sh
# opzione A: mirror (tkn corrente, con tkn + opc + tkn-pac)
$ wget https://mirror.openshift.com/pub/openshift-v4/clients/pipeline/latest/tkn-linux-amd64.tar.gz
$ tar xf tkn-linux-amd64.tar.gz tkn && ./tkn version

# opzione B: client OCP (tkn incluso)
$ wget https://mirror.openshift.com/pub/openshift-v4/clients/ocp/4.21.0/openshift-client-linux.tar.gz
$ tar xf openshift-client-linux.tar.gz tkn oc
```

## 3. Creare i project

- `devops`: qui vivono i Task, la Pipeline, l'EventListener, la PVC, Nexus.
- `project-target-test`: qui la pipeline builda/spinge le immagini e deploya l'app.

```sh
$ oc new-project devops
$ oc new-project project-target-test
```

## 4. Deployare il server Nexus

```sh
SC=$(oc get sc -o name --show-sigs | head -1 | xargs)   # StorageClass del cluster (es. thin-csi)
curl -s https://raw.githubusercontent.com/gmeghnag/tekton-lab/main/templates/nexus3.yaml \
  | oc process -f - -n devops -p PVC_SIZE=10Gi -p SC_NAME=$SC | oc apply -n devops -f -
```

- Il template è in `template.openshift.io/v1` (la vecchia forma `apiVersion: v1 / kind: Template`
  non è più risolta da `oc process`).
- In 4.14+ il `DeploymentConfig` è deprecato ma ancora disponibile: la warning è cosmetica.

### Creare i repository Maven su Nexus
- `maven-group` (proxy a Maven Central + aggregatore), `maven-releases`, `maven-snapshots`.
- https://blog.sonatype.com/using-nexus-3-as-your-repository-part-1-maven-artifacts

### Onboarding Nexus 3.9x (EULA + telemetria) — obbligatorio
Su Nexus 3.9x, finché non si completa l'onboarding **tutte** le richieste di risoluzione
artifact rispondono `403 You must accept the End User License Agreement (EULA)...`.

- L'EULA **non si accetta via REST API** sui 3.9x (i path `/service/rest/v1/onboarding*`,
  `/eula`, `/security/onboarding` sono `404`): si completa solo col **wizard UI**
  (apri il route di Nexus, login `admin` + password temporanea in `/nexus-data/admin.password`).
- Il wizard **forza il cambio della password admin** come primo step: la password temporanea
  (UUID) smette di funzionare subito dopo. Impostarne una nuova e registrarla nel
  `settings.xml` dell'app.
- Step successivi: accetta l'EULA, lascia **anonymous access disabilitato** (richiede auth
  su tutte le repo), conferma la **mandatory base telemetry**. Dopo il "Finish" la
  risoluzione torna `200` e il health check `Baseline Telemetry` diventa `Active`.
- **Re-deploy**: i re-run della pipeline deployano lo stesso `version` (es.
  `2.1.0-<hash-commit>`), quindi `maven-releases` va messo in `writePolicy: ALLOW`
  (default `ALLOW_ONCE` risponde `409`). Su 3.9x il `PATCH` sul repository è `405`:
  ```sh
  $ curl -u admin:<pw> $NX/service/rest/v1/repositories/maven/hosted/maven-releases \
      | python3 -c "import json,sys; d=json.load(sys.stdin); d['storage']['writePolicy']='ALLOW'; print(json.dumps(d))" > /tmp/r.json
  $ curl -u admin:<pw> -X PUT -H 'Content-Type: application/json' --data @/tmp/r.json \
      $NX/service/rest/v1/repositories/maven/hosted/maven-releases
  ```

### Route Nexus: doppia route (HTTPS per il browser, HTTP per Maven)
Sul router di questo cluster un route **con TLS non viene servito su HTTP** (la mappa
`os_http_be.map` del frontend HTTP resta vuota per lui): se aggiungi `spec.tls` al route
principale per usare la UI in HTTPS, la porzione `http://` risponde `503 application is not
available` — e la pipeline si rompe, perché `mvn deploy` usa ancora `http://`.

La soluzione è **due route** sullo stesso Service `nexus3`:

- **`nexus3`** (UI, browser): terminazione `edge`. Si crea con
  ```sh
  $ oc patch route nexus3 -n devops --type merge -p \
      '{"spec":{"tls":{"termination":"edge","insecureEdgeTerminationPolicy":"None"}}}'
  ```
  → `https://nexus3-devops.apps.<domain>/` (200). L'HTTP sulla stessa host resta 503,
  ma non viene usato.
- **`nexus3-http`** (Maven, pipeline): senza TLS, host dedicato
  `nexus3-http-devops.apps.<domain>`. È il **mirror** da mettere in `settings.xml` e nei
  repository di `pom.xml`:
  ```sh
  $ oc apply -n devops -f - <<'EOF'
  apiVersion: route.openshift.io/v1
  kind: Route
  metadata:
    name: nexus3-http
    labels:
      app: nexus3
  spec:
    host: nexus3-http-devops.apps.<domain>
    port:
      targetPort: 8081-tcp
    to:
      kind: Service
      name: nexus3
  EOF
  ```

> Nota: se invece vuoi una sola route HTTP (senza HTTPS per la UI), basta lasciare il
> route del template invariato (nessun `spec.tls`) e usarlo sia dal browser sia da Maven.

## 5. PVC della pipeline

```sh
$ oc apply -n devops -f pvc.yaml        # pvc-pipeline-demo, 4Gi, RWO
```

## 6. Creare le risorse Tekton in `devops`

```sh
$ oc apply -k https://github.com/gmeghnag/tekton-lab -n devops

pipeline.tekton.dev/pipeline-demo created
task.tekton.dev/clean-volume created
task.tekton.dev/clone-repository created
task.tekton.dev/maven-deploy created
task.tekton.dev/buildah created
task.tekton.dev/deploy-app created
eventlistener.triggers.tekton.dev/pipeline-demo-el created
triggerbinding.triggers.tekton.dev/pipeline-demo-tb created
triggertemplate.triggers.tekton.dev/pipeline-demo-tt created
```

Verifica che l'EventListener sia pronto:

```sh
$ oc get eventlistener -n devops pipeline-demo-el
$ oc get pods -n devops -l app.kubernetes.io/part-of=tekton-triggers
```

### Risorse create
 - **task.tekton.dev/clean-volume** — pulisce la PVC usata per persistere dati tra i Task.
 - **task.tekton.dev/clone-repository** — clona il sorgente, legge la versione da `pom.xml`,
   la fa seguire dall'hash git e la espone come result `ARTIFACT_VERSION`.
 - **task.tekton.dev/maven-deploy** — `mvn deploy` (JDK 8) per pushare l'`.jar` su Nexus.
 - **task.tekton.dev/buildah** — build (privileged, storage-driver `vfs`) e push dell'immagine
   nel project target. Anche lo step `push` è `privileged` (l'utente default di buildah
   richiede user namespace per leggere lo storage condiviso tra step).
 - **task.tekton.dev/deploy-app** — `oc new-app <project>/<image>:<tag>` (crea ImageStream +
   Deployment + Service, idempotente), tagga l'ImageStream come `latest` ed espone il
   service con una route (`oc expose service/<app>`, no-op se già presente).
 - **eventlistener / triggerbinding / triggertemplate** — trigger su `push` a `refs/heads/main`.

## 7. ServiceAccount e permessi

`pipeline` è creato dall'operatore. Crea `build-bot` (serve a buildah per spingere immagini)
e dai i ruoli nel project target:

```sh
oc create sa build-bot -n devops
oc adm policy add-scc-to-user privileged -z build-bot -n devops
oc adm policy add-role-to-user system:image-builder system:serviceaccount:devops:pipeline -n project-target-test
oc adm policy add-role-to-user image-pusher       system:serviceaccount:devops:build-bot -n project-target-test
oc adm policy add-role-to-user edit               system:serviceaccount:devops:build-bot -n project-target-test
oc adm policy add-role-to-user edit               system:serviceaccount:devops:pipeline -n project-target-test
```

> `image-pusher` è richiesto a `build-bot` (in 4.x `edit` da solo non basta per `imagestreams`).

## 8. Avviare la pipeline

- **manualmente con `tkn`**
- **manualmente creando un `PipelineRun`** (vedi `pipelinerun.yaml`)
- **automaticamente** a ogni `push` sul repo (configura il webhook GitHub sull'EventListener)

```sh
$ oc apply -n devops -f pipelinerun.yaml
$ tkn pipeline start pipeline-demo -n devops \
    -p REPOSITORY_URL=https://github.com/<you>/<app>.git \
    -p REGISTRY_URL=image-registry.openshift-image-registry.svc:5000 \
    -p BUILD_IMAGE=<app> -p OCP_PROJECT=project-target-test \
    -p BRANCH_NAME=main -p COMMIT_ID=HEAD
$ tkn pipelinerun logs -n devops -f
```

---

## Note di migrazione 2021 → 1.23

Cambiamenti applicati per rendere la POC eseguibile su OSP 1.23 (verificati sulle codebase
ufficiali `tektoncd/pipeline@v1.12.x`, `tektoncd/triggers@v0.36.x`, `openshift-pipelines/tektoncd-pipeline@release-v1.23.x`):

- **API `tekton.dev/v1alpha1` rimossa** → Task/Pipeline/PipelineRun portate a `tekton.dev/v1`.
- **`PipelineRun.spec.serviceAccountName` e `serviceAccountNames` rimossi**. L'estensione OpenShift
  `serviceAccountNames` (override per-taskName) non esiste più nel fork 1.23. Sostituiti con
  `spec.taskRunTemplate.serviceAccountName` (default per tutti i Task) +
  `spec.taskRunSpecs[].{pipelineTaskName, serviceAccountName}` (override per singolo Task).
- **Triggers 0.36**: l'interceptor CEL inline (`interceptors: [{cel: {filter, overlays}}]`) è
  sparito dal tipo Go. Ora è un *builtin* referenziato con `interceptors: [{ref: {name: cel},
  params: [{name: filter, ...}, {name: overlays, ...}]}]`. `template.name` → `template.ref`.
  `TriggerBinding`/`TriggerTemplate` in `triggers.tekton.dev/v1beta1`.
- **Immagini rimpiazzate/pinnate** (le `:latest` 2021 erano morte o incompatibili):
  - `docker.io/maven` → `maven:3.8-openjdk-8` (l'app è Spring Boot 2.3.3 / Java 1.8; il `maven:latest`
    moderno è JDK 21+ e fallirebbe).
  - `quay.io/buildah/stable:latest` → `buildah/stable:v1.38` (i flag usati `--storage-driver`,
    `bud --format=oci --no-cache --tls-verify` e `push --tls-verify --digestfile` sono tutti ancora validi).
  - `quay.io/openshift/origin-cli:latest` → `origin-cli:4.21` (`origin-cli` oggi ospita il client `oc`
    moderno; `project/tag/new-app/expose/get` tutti presenti).
  - `sonatype/nexus3` → `nexus3:3.98.1`.
  - `docker.io/alpine/git` (clean/clone) ancora valido, non toccato.

### App di esempio: fork e fix (eseguiti, validati)
Il completamento end-to-end (maven→buildah→deploy) dipende dall'app di esempio, che nel 2021
aveva hardcoded un Nexus morto e un'immagine morta. Sono stati creati:

- **Fork** `gmeghnag/spring-petclinic` (da `GMH501/spring-petclinic`), con:
  - `settings.xml` mirror + `pom.xml` (`repositories`, `distributionManagement`) ripuntati al
    Nexus del cluster **sulla route HTTP dedicata** (es.
    `nexus3-http-devops.apps.<domain>/repository/maven-group|releases|snapshots`) e
    credenziali `admin`/`<pw-post-onboarding>`.
  - **id server `central` in `settings.xml`**: il mirror ha `id: central` e `mirrorOf: *`, e
    Maven matcha le credenziali per **id repository/server**: senza un `<server id="central">`
    il download risponde `401 Unauthorized` (i soli `nexus-snapshots`/`nexus-releases` non
    bastano).
  - `Dockerfile`: `FROM docker.io/openjdk` → `FROM docker.io/eclipse-temurin:8` (il tag `latest`
    di `openjdk` è stato rimosso da Docker Hub; l'app è Java 1.8).

### Fix dei Task emersi nel run end-to-end (OCP 4.21)
- **`tasks/build-image.yaml`**: lo step `push` (e non solo `build`) richiede
  `securityContext: {privileged: true}`: l'utente default dell'immagine `buildah/stable:v1.38`
  non è root, e senza privilegi buildah non può scrivere `/proc/*/uid_map` per usare lo
  storage condiviso tra step → `error writing "0 0 4294967295\n" to /proc/22/uid_map:
  operation not permitted`.
- **`tasks/deploy-app.yaml`**: su OCP 4.x non esiste un ImageStream pre-esistente, quindi
  `oc tag <app>:<ver> <app>:latest` (logica 2021) fallisce. Ora il task usa
  `oc new-app <project>/<image>:<tag> --name <app>` (crea ImageStream + Deployment +
  Service, idempotente), `oc tag ... latest` ed infine
  `oc expose service/<app>` (con check: no-op se la route esiste già), così l'app resta
  raggiungibile anche da fuori il cluster.

### Risultato (validato su OCP 4.21.37 / OSP 1.23.4)
`pipeline-demo` **Succeeded** end-to-end: clean-volume → clone-repository → maven-deploy
(`BUILD SUCCESS`, artifact `2.1.0-<hash>` in `maven-releases` su Nexus) → buildah (build da
`eclipse-temurin:8` + push su `image-registry.../project-target-test/spring-petclinic:2.1.0-<hash>`)
→ deploy-app (Deployment 1/1 Available, service esposto con route; app risponde HTTP 200 su
`/` e `/owners`, sia in-cluster sia via `http://spring-petclinic-project-target-test.apps.<domain>/`).
