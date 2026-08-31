# SLSA Level 1 Implementation POC — Proofpoint (container / Jib)

Read `SLSA-Feynman.md` first if you have not. SLSA Level 2 is covered separately in `SLSA-L2-POC-Proofpoint.md` — do not start it until every item in the L1 checklist at the bottom of this document is green.

This fork adapts the original JAR lab so **provenance receipts the container image**, which is how Proofpoint Java services typically publish (Jib, no local Docker daemon).

The community **SLSA Provenance Attestation** plugin (`slsa`) remains a valid L1 tool **when the shipped artifact is a file on disk** (a JAR). It is the **wrong** tool for Jib: `provenanceRecorder` hashes file bytes. Pointing it at `target/jib-image.digest` would attest “hash of a short text file,” not the **image manifest digest** Jib wrote inside that file.

---

## How this POC matches the Proofpoint environment

Everything below was verified from the two support bundles (controllers `buildrel` and `cloud15`, captured 2026-08-25). This container lab does **not** install the `slsa` plugin. Pipeline Groovy (`readFile` / `writeFile` / `archiveArtifacts`) is already on the controllers.

| Environment fact (from support bundles) | How the lab uses it |
|---|---|
| Jobs are overwhelmingly Pipeline: 5,662 `WorkflowJob` + 90 Multibranch on `buildrel`; 1,462 + 151 on `cloud15` | POC is a Multibranch Pipeline with a source-controlled Jenkinsfile |
| GitHub Branch Source `1925.x`, `cloudbees-github-reporting`, plus Bitbucket Server integration installed | Lab 1 uses GitHub branch source; a Bitbucket note is included |
| Two agent models: Kubernetes pods (kubernetes plugin `4419.x`, namespace `cloudbees-agents`, service account `jenkins-agents`) **and** EC2 cloud `ec2-instances-cloud` | Primary Jenkinsfile uses a Kubernetes pod agent; §2.5 gives the EC2 label variant. The provenance stage uses Jenkins steps (`readFile`, `writeFile`, `archiveArtifacts`) so it runs on either |
| Default pod agent image is pulled through the internal mirror `repocache.nonprod.ppops.net/docker/...` | The lab Jenkinsfile defaults to the public image so it runs anywhere; when running inside Proofpoint, switch it to the `repocache` mirror path |
| The `slsa` plugin is **not** in either bundle's plugin list | Leave it uninstalled for this container lab. Use it later only for JAR-only pipelines |
| `pipeline-utility-steps 2.20.0` installed | Available for ad-hoc digest checks; Lab 3 uses built-in `readFile` |
| `artifact-manager-s3 962.x` installed; S3 blob store configured but the artifact-manager factory list in the bundle snapshot is **empty** | `archiveArtifacts` + `fingerprint: true` works on controller disk or S3. §3.5 tells you how to confirm which one is active |
| Fingerprint storage: local file storage (default), cleanup enabled | Fingerprints UI is the L1 audit trail for `provenance-l1.json` and `jib-image.digest` |
| Implicit shared library `pfpt-ci-pipeline` (`PFPT-Internal/ieng-pfpt-ci-pipeline`) | This lab keeps steps in the Jenkinsfile so you can see them. Copy-paste `vars/recordJibSlsaL1` snippet at the bottom when you fold this into the library |

> **Note on anonymized names:** support bundles anonymize job names, labels, and agent names. Where the lab says *"your EC2 label"* or *"your team folder"*, substitute the real value from the controller UI.

> **Image sourcing:** the lab Jenkinsfile defaults to the public `maven:3.9-eclipse-temurin-17` (Docker Hub) so it runs on any test controller. `repocache.nonprod.ppops.net` is Proofpoint's **internal-only** mirror. When delivering inside Proofpoint, switch the Maven **and** Jib `from` image to `repocache` paths. Jib `from` in this `pom.xml` is `eclipse-temurin:17-jre` (Docker Hub) for the same reason.

---

## What SLSA Level 1 requires (and does not)

Level 1 = **the build process is fully scripted and produces provenance**. Concretely, for every build:

1. The pipeline definition lives in SCM (no UI-defined build steps).
2. The build runs on the hosted platform (CloudBees CI), not a laptop.
3. The build outputs a provenance document: what was built, from which commit, by which builder, and the artifact's digest.

No signing, no tamper-proofing — that is Level 2. L1 is the auditable paper trail.

```
Lab 1:  Create the Multibranch Pipeline job                     (~15 min)
Lab 2:  Base Jenkinsfile — Maven build + test                   (~20 min)
Lab 3:  L1 provenance — Jib digest + provenance-l1.json         (~25 min)
```

---

## Pre-lab — repo and access

1. Use this fork (or fork it again): the GitHub repo that contains this `Jenkinsfile` and `pom.xml` Jib config. A stock Spring PetClinic tree without Jib will not write `target/jib-image.digest`.
2. Confirm you can log in to the agreed nonprod controller (SAML SSO) and can create items in a test folder.
3. (Proofpoint delivery only) Confirm build agents can reach the internal registry mirror `repocache.nonprod.ppops.net`. On a test controller outside Proofpoint, skip this; the lab uses public images.

After `./mvnw -B -DskipTests package jib:buildTar` the example app produces:

```
target/jib-image.digest   ← image manifest digest (sha256:<64 hex>) — L1 subject
target/jib-image.tar      ← image tarball (not archived by default; large)
target/bom.json / target/bom.xml   ← CycloneDX SBOM
```

The lab uses **`jib:buildTar`** so a registry is not required. When Proofpoint already pushes with Jib, swap that goal for `jib:build` (same digest file). Do not use `jib:dockerBuild` — that needs a Docker daemon, which this pattern avoids.

> For Proofpoint services whose poms do **not** declare CycloneDX, invoke it by full coordinates:
> `./mvnw -B org.cyclonedx:cyclonedx-maven-plugin:2.8.0:makeAggregateBom`

---

## Lab 1 — Create the Multibranch Pipeline

**Goal:** Connect the controller to the fork so every push triggers a build. This satisfies the "hosted build service" foundation of SLSA.

### Steps

**1.1 Credential**

1. On the controller: **Manage Jenkins → Credentials → System → Global credentials → Add Credentials**
2. Kind **Username with password** — GitHub username + a classic PAT with the full **`repo`** scope
3. ID: `github-slsa-poc`

> Proofpoint also runs Bitbucket Server integration on this controller. If the pilot repo lives in Bitbucket, use **Bitbucket** as the branch source in 1.2 with the existing Bitbucket endpoint; everything else in this POC is identical.

**1.2 Job**

1. In your team/test folder: **New Item** → name `slsa-poc-petclinic` → **Multibranch Pipeline**
2. **Branch Sources → Add source → GitHub**: credential `github-slsa-poc`, repository HTTPS URL of your fork
3. **Build Configuration**: Mode = by Jenkinsfile, Script Path = `Jenkinsfile`
4. Save — the scan runs and reports `'Jenkinsfile' not found`, which is expected until Lab 2.

### Expected result

Multibranch job exists inside a folder, scan completes successfully with no buildable branches yet.

---

## Lab 2 — Base Jenkinsfile (matches the Proofpoint agent model)

**Goal:** A scripted, source-controlled Maven build — the core L1 requirement.

### 2.1 The Jenkinsfile — Kubernetes pod agent (primary)

This mirrors how pod agents already run on `buildrel` (namespace `cloudbees-agents`, service account `jenkins-agents` — both injected by the controller's defaults, so the Jenkinsfile does not declare them). Create `Jenkinsfile` at the repo root:

```groovy
pipeline {
    agent {
        kubernetes {
            yaml """
apiVersion: v1
kind: Pod
spec:
  containers:
  - name: maven
    # Public image (Docker Hub). Inside Proofpoint, switch to the internal
    # mirror: repocache.nonprod.ppops.net/docker/maven:3.9-eclipse-temurin-17
    image: maven:3.9-eclipse-temurin-17
    command: [sleep, infinity]
    env:
    - name: MAVEN_OPTS
      value: "-Xmx512m"
    volumeMounts:
    - name: maven-cache
      mountPath: /root/.m2
  volumes:
  - name: maven-cache
    emptyDir: {}
"""
        }
    }

    options {
        timeout(time: 30, unit: 'MINUTES')
        disableConcurrentBuilds()
    }

    stages {

        stage('Checkout') {
            steps {
                checkout scm
                script {
                    env.GIT_COMMIT_SHA = sh(script: 'git rev-parse HEAD', returnStdout: true).trim()
                    env.GIT_REPO_URL   = sh(script: 'git remote get-url origin', returnStdout: true).trim()
                    echo "Building commit: ${env.GIT_COMMIT_SHA}"
                }
            }
        }

        stage('Build') {
            steps {
                container('maven') {
                    sh './mvnw -B -DskipTests package org.cyclonedx:cyclonedx-maven-plugin:2.8.0:makeAggregateBom'
                }
            }
        }

        stage('Test') {
            steps {
                container('maven') {
                    sh './mvnw -B test'
                }
            }
            post {
                always {
                    junit testResults: 'target/surefire-reports/*.xml', allowEmptyResults: true
                }
            }
        }

    }

    post {
        success { echo "Build succeeded: ${env.JOB_NAME} #${env.BUILD_NUMBER}" }
        failure { echo "Build failed: check the stage logs above" }
    }
}
```

Notes on why this differs from a generic lab Jenkinsfile:

- **Image source** — the lab default is the public `maven:3.9-eclipse-temurin-17` so the pipeline runs anywhere. When running inside Proofpoint, switch to `repocache.nonprod.ppops.net/docker/maven:3.9-eclipse-temurin-17`. Docker Hub pulls may be blocked there; `repocache` is internal-only.
- **No `namespace` or `serviceAccountName` in the pod YAML — deliberately.** The Proofpoint controllers inject `cloudbees-agents` / `jenkins-agents` via JVM flags. Hardcoding a namespace makes the pipeline fail with `pods is forbidden ... 403` on controllers whose service account has RBAC elsewhere.

### 2.5 Variant — EC2 agent

Many Proofpoint workloads run on the `ec2-instances-cloud` EC2 templates rather than pods. The same pipeline works by swapping the agent block and dropping the `container()` wrappers:

```groovy
agent { label 'YOUR-EC2-TEMPLATE-LABEL' }   // e.g. the m6a.large standard-network label
```

Requirements on the AMI: Java 17+ for `./mvnw`. Lab 3 uses Jenkins steps (`readFile`, `writeFile`, `archiveArtifacts`), not `sha256sum` or `cosign`, so it behaves the same on pods and EC2.

### 2.6 Commit, push, build

```bash
git add Jenkinsfile
git commit -m "SLSA L1 POC: scripted Maven build"
git push origin main
```

Trigger **Scan Multibranch Pipeline Now**; the `main` pipeline appears and builds. First build takes 3–5 minutes (cold Maven cache).

### Expected result

Green build with Checkout / Build / Test stages; log shows the pod (or EC2 agent) provisioning, `./mvnw` producing the JAR, and tests passing.

---

## Lab 3 — SLSA Level 1 provenance (Jib image digest)

**Goal:** Every build archives a fingerprinted Jib digest file, an SBOM, and a provenance JSON whose `subject.digest.sha256` **equals the hex inside** `target/jib-image.digest` (Jib’s image manifest digest), plus git commit and builder URLs.

### Why not `provenanceRecorder` on the digest file

Jib writes `target/jib-image.digest` as a single line: `sha256:` plus 64 hex characters. That **line is the image digest**.

The `slsa` plugin’s `provenanceRecorder` hashes **whatever files match `artifactFilter`**. If you set `artifactFilter: 'target/jib-image.digest'`, the attestation subject is SHA-256 of those **file bytes** (a hash of the text `sha256:abcd…`), which is a different value from `abcd…`. Auditors comparing provenance to `image@sha256:…` in the registry would see a mismatch.

Correct approach: **read** the file, strip the `sha256:` prefix, put that hex on `subject.digest.sha256`. Set `subject.name` from the image reference you already passed to Jib (`JIB_IMAGE` / `pom.xml` `<jib.to.image>`). The digest file does not include the image name ([Jib issue 4348](https://github.com/GoogleContainerTools/jib/issues/4348)).

JAR-only services can still use `provenanceRecorder` on `target/*.jar`. Do not mix that glob with a Jib image publish.

### 3.1 Jib in the pom (already in this fork)

`jib-maven-plugin` `3.5.2` is configured with `<to><image>${jib.to.image}</image></to>`. It is **not** bound to `package`, so a normal `./mvnw package` does not push. The pipeline calls `jib:buildTar` explicitly.

Inside Proofpoint, change the Jib `<from><image>` (and the Maven agent image) to `repocache.nonprod.ppops.net/docker/...` paths. To push to their registry instead of a tarball, replace `jib:buildTar` with `jib:build` and set `-Djib.to.image=registry.example/…`.

### 3.2 Add Jib to Build; archive the digest; write provenance

The repo-root `Jenkinsfile` is the Lab 3 deliverable. Differences from Lab 2:

- `environment { JIB_IMAGE = 'spring-petclinic:4.0.0-SNAPSHOT' }` — must match the pom property.
- Build also runs `jib:buildTar`.
- Archive fingerprints `target/jib-image.digest` (not the JAR as the L1 subject).
- Provenance stage reads that file, writes `provenance-l1.json` at the **workspace root** (writable by the `jnlp` user; `target/` is often root-owned on pod agents).

See the complete file in §3.3 (also `Jenkinsfile` in this repository).

### 3.3 Complete Jenkinsfile after Lab 3

Use the `Jenkinsfile` at the repository root. The provenance stage is:

```groovy
stage('SLSA L1 Provenance') {
    steps {
        script {
            def digestPath = 'target/jib-image.digest'
            if (!fileExists(digestPath)) {
                error("Jib digest file missing: ${digestPath}. The Build stage must run jib:buildTar or jib:build first.")
            }
            def raw = readFile(digestPath).trim()
            if (!(raw ==~ /sha256:[a-fA-F0-9]{64}/)) {
                error("Unexpected Jib digest (want sha256: plus 64 hex chars): '${raw}'")
            }
            def imageSha256 = raw.substring('sha256:'.length()).toLowerCase()
            // ... write provenance-l1.json with subject.digest.sha256 = imageSha256
        }
        archiveArtifacts artifacts: 'provenance-l1.json', fingerprint: true
    }
}
```

`writeFile` must stay at workspace-root `provenance-l1.json`, not under `target/` — same uid rule as the old `slsa/` directory: Maven runs as root, `jnlp` is `jenkins`.

### 3.4 Commit, push, run

```bash
git add Jenkinsfile pom.xml SLSA-L1-POC.md
git commit -m "SLSA L1: Jib digest provenance (container subject)"
git push origin main
```

### 3.5 Inspect the results

1. **Archived Artifacts** on the build page: `jib-image.digest`, `bom.json` / `bom.xml`, and `provenance-l1.json`.
   - Check **Manage Jenkins → System → Artifact Management for Builds** for disk vs S3. Either backend satisfies L1.
2. Open `provenance-l1.json` and `jib-image.digest`. Confirm:
   - `subject.digest.sha256` equals the 64 hex characters **after** `sha256:` in the digest file (not a hash of the digest file),
   - `source.commit` matches `GIT_COMMIT_SHA` in the build log,
   - `builder.build_url` points at this run.
3. **Fingerprints** in the build’s left nav — the digest file and `provenance-l1.json` both appear.

### Expected result

Five-stage green build. Every subsequent build produces image digest + SBOM + provenance JSON with no human step in the loop.

---

## SLSA Level 1 completion checklist

- [ ] Jenkinsfile in SCM; no UI-defined build steps (Lab 2)
- [ ] Build runs on the CloudBees CI controller via pod or EC2 agent — never locally (Lab 2)
- [ ] Jib ran without a Docker daemon (`jib:buildTar` or `jib:build`) (Lab 3)
- [ ] `target/jib-image.digest` archived with `fingerprint: true` (Lab 3)
- [ ] CycloneDX SBOM (`bom.json`/`bom.xml`) archived per build (Lab 3)
- [ ] `provenance-l1.json` archived and fingerprinted; `subject.digest.sha256` matches the hex inside the digest file; git commit + `BUILD_URL` present (Lab 3)
- [ ] Fingerprints visible in the Jenkins Fingerprints UI (Lab 3)
- [ ] Verified whether archived artifacts land on controller disk or the S3 bucket (§3.5)

**Definition of done for the proposal milestone:** one Proofpoint-shaped Maven/Jib pipeline on the agreed controller runs this end-to-end, and the team can locate the provenance for any given build in under a minute.

Proceed to `SLSA-L2-POC-Proofpoint.md` (target: September 22) to add signing. Cosign must be **baked into the agent image** (air-gapped / `repocache`); do not download it during the build. Use `--tlog-upload=false` until a self-hosted transparency log exists.

---

## Copy-paste for `pfpt-ci-pipeline` (later)

Not required to finish this lab. After Jib publish, a shared-library vars step can wrap the same parse/write/archive logic:

```groovy
// vars/recordJibSlsaL1.groovy
def call(Map cfg = [:]) {
    def digestPath = cfg.digestPath ?: 'target/jib-image.digest'
    def imageName = cfg.imageName ?: error('recordJibSlsaL1: imageName is required')
    if (!fileExists(digestPath)) {
        if (cfg.skipIfMissing) {
            echo "recordJibSlsaL1: skip, missing ${digestPath}"
            return
        }
        error("recordJibSlsaL1: missing ${digestPath}")
    }
    def raw = readFile(digestPath).trim()
    if (!(raw ==~ /sha256:[a-fA-F0-9]{64}/)) {
        error("recordJibSlsaL1: bad digest '${raw}'")
    }
    def imageSha256 = raw.substring('sha256:'.length()).toLowerCase()
    env.GIT_COMMIT_SHA = env.GIT_COMMIT_SHA ?: sh(script: 'git rev-parse HEAD', returnStdout: true).trim()
    env.GIT_REPO_URL = env.GIT_REPO_URL ?: sh(script: 'git remote get-url origin', returnStdout: true).trim()
    writeFile file: (cfg.outputFile ?: 'provenance-l1.json'), text: """{
  "slsa_level": "1",
  "subject": { "name": "${imageName}", "digest": { "sha256": "${imageSha256}" } },
  "source": { "repository": "${env.GIT_REPO_URL}", "commit": "${env.GIT_COMMIT_SHA}" },
  "builder": { "id": "${env.JENKINS_URL}", "build_url": "${env.BUILD_URL}" }
}
"""
    archiveArtifacts artifacts: (cfg.outputFile ?: 'provenance-l1.json'), fingerprint: true
}
```

Call after existing `publish()` / Jib: `recordJibSlsaL1(imageName: env.JIB_IMAGE)`. Do not invent `publish()` internals in this lab.

---

## Troubleshooting

**Pod never starts / `ErrImagePull` on the maven container**
- Outside Proofpoint: use public `maven:3.9-eclipse-temurin-17`. `repocache.nonprod.ppops.net` does not resolve on the public internet.
- Inside Proofpoint: use `repocache.nonprod.ppops.net/docker/maven:3.9-eclipse-temurin-17`; ask the registry team to prefetch rather than falling back to Docker Hub.

**Jib cannot pull `eclipse-temurin:17-jre`**
Same mirror rule as the Maven agent. Set `<from><image>repocache.nonprod.ppops.net/docker/eclipse-temurin:17-jre</image></from>` (or whatever tag their cache uses).

**`ERROR: Failed to launch ...` with `pods is forbidden` (403)**
Remove `metadata.namespace` and `serviceAccountName` from the pod YAML.

**`Jib digest file missing`**
Build did not run `jib:buildTar` or `jib:build`, or Jib `outputPaths.digest` was overridden to another path. Confirm `target/jib-image.digest` in the workspace.

**`Unexpected Jib digest`**
File is empty, has a trailing path, or is `image@sha256:…` (not what current Jib writes). This lab expects exactly `sha256:` + 64 hex.

**Provenance `sha256` does not match the registry**
You hashed the digest **file** (plugin or `sha256sum target/jib-image.digest`). Read the file contents instead.

**`AccessDeniedException` writing under `target/`**
Write `provenance-l1.json` at the workspace root. Maven created `target/` as root; `jnlp` cannot mkdir there.

**`./mvnw: Permission denied`**
`git update-index --chmod=+x mvnw && git commit -m "make mvnw executable" && git push`

**Maven cannot resolve `jib-maven-plugin` or `cyclonedx-maven-plugin`**
The agent is not using the Artifactory / `repocache` Maven mirror. Check `~/.m2/settings.xml` (`config-file-provider` is installed on the audited controllers).

**Builds queue forever on the EC2 variant**
The label must match an `ec2-instances-cloud` template exactly (labels in the support bundle are anonymized).
