// SLSA Level 1 POC — container/Jib (see SLSA-L1-POC.md Lab 3)
// No Docker daemon. jib:buildTar writes target/jib-image.digest (image manifest digest).
// Do not point the slsa plugin at that file — it would hash the text file, not the image.
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
    # (repocache is internal-only; it does not resolve outside their network)
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

    environment {
        // Must match pom.xml <jib.to.image> — this is the provenance subject name.
        JIB_IMAGE = 'spring-petclinic:4.0.0-SNAPSHOT'
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
                    // package + SBOM, then Jib tarball (writes target/jib-image.digest without a registry)
                    sh './mvnw -B -DskipTests package org.cyclonedx:cyclonedx-maven-plugin:2.8.0:makeAggregateBom jib:buildTar'
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

        stage('Archive') {
            steps {
                archiveArtifacts artifacts: 'target/jib-image.digest', fingerprint: true
                archiveArtifacts artifacts: 'target/bom.json, target/bom.xml', allowEmptyArchive: true
            }
        }

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
                    def imageName = (env.JIB_IMAGE ?: '').replace('\\', '\\\\').replace('"', '\\"')
                    def repoUrl = (env.GIT_REPO_URL ?: '').replace('\\', '\\\\').replace('"', '\\"')
                    def commit = (env.GIT_COMMIT_SHA ?: '').replace('\\', '\\\\').replace('"', '\\"')
                    def builderId = (env.JENKINS_URL ?: '').replace('\\', '\\\\').replace('"', '\\"')
                    def buildUrl = (env.BUILD_URL ?: '').replace('\\', '\\\\').replace('"', '\\"')
                    def provenance = """{
  "slsa_level": "1",
  "subject": {
    "name": "${imageName}",
    "digest": { "sha256": "${imageSha256}" }
  },
  "source": {
    "repository": "${repoUrl}",
    "commit": "${commit}"
  },
  "builder": {
    "id": "${builderId}",
    "build_url": "${buildUrl}"
  }
}
"""
                    writeFile file: 'provenance-l1.json', text: provenance
                    echo "SLSA L1 subject ${env.JIB_IMAGE} sha256:${imageSha256}"
                }
                archiveArtifacts artifacts: 'provenance-l1.json', fingerprint: true
            }
        }

    }

    post {
        success { echo "SLSA L1 build complete: ${env.JOB_NAME} #${env.BUILD_NUMBER}" }
        failure { echo "Build failed: check the stage logs above" }
    }
}
