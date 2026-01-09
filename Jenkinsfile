#!/bin/env groovy

pipeline {
    agent {
        label "master"
    }
    tools {
        // Note: this should match with the tool name configured in your jenkins instance (JENKINS_URL/configureTools/)
        maven "maven"
        jdk "jdk11"
    }
    environment {
        // This can be nexus3 or nexus2
        NEXUS_VERSION = "nexus3"
        // This can be http or https
        NEXUS_PROTOCOL = "https"
        // Where your Nexus is running
        NEXUS_URL = "nexus.united-portal.com"
        // Repository where we will upload the artifact
        NEXUS_REPOSITORY = "maven-snapshots"
        // Jenkins credential id to authenticate to Nexus OSS
        NEXUS_CREDENTIAL_ID = "nexus-jenkins-credentials"
        // Repository where we will upload the docker container
        NEXUS_DOCKER_PRIVATE_REGISTRY = "private-registry.united-portal.com"
    }
    stages {
        /* stage("clone code") {
            steps {
                script {
                    // Let's clone the source
                    git 'https://github.engineering.zhaw.ch/bacn/spring-petclinic-maven-java11';
                }
            }
        } */
        stage("mvn build, test, install ") {
            steps {
                script {
                    // If you are using Windows then you should use "bat" step
                    // Since unit testing is out of the scope we skip them
                    sh "mvn -B clean install"
                    junit "**/target/surefire-reports/*.xml"
                }
            }
        }
        stage("build & SonarQube analysis") {
            steps {
                script {
                    withSonarQubeEnv(installationName: 'sonar-united-portal-server', credentialsId: 'SonarQubeToken')  {
                        sh "mvn sonar:sonar"
                    }
                }
            }
        }
        stage("publish maven to nexus") {
            steps {
                script {
                    // Read POM xml file using 'readMavenPom' step , this step 'readMavenPom' is included in: https://plugins.jenkins.io/pipeline-utility-steps
                    pom = readMavenPom file: "pom.xml";
                    // Find built artifact under target folder
                    filesByGlob = findFiles(glob: "target/*.${pom.packaging}");
                    // Print some info from the artifact found
                    echo "${filesByGlob[0].name} ${filesByGlob[0].path} ${filesByGlob[0].directory} ${filesByGlob[0].length} ${filesByGlob[0].lastModified}"
                    // Extract the path from the File found
                    artifactPath = filesByGlob[0].path;
                    // Assign to a boolean response verifying If the artifact name exists
                    artifactExists = fileExists artifactPath;
                    if(artifactExists) {
                        echo "*** File: ${artifactPath}, group: ${pom.groupId}, packaging: ${pom.packaging}, version ${pom.version}";
                        nexusArtifactUploader(
                                nexusVersion: NEXUS_VERSION,
                                protocol: NEXUS_PROTOCOL,
                                nexusUrl: NEXUS_URL,
                                groupId: pom.groupId,
                                version: pom.version,
                                repository: NEXUS_REPOSITORY,
                                credentialsId: NEXUS_CREDENTIAL_ID,
                                artifacts: [
                                        // Artifact generated such as .jar, .ear and .war files.
                                        [artifactId: pom.artifactId,
                                         classifier: '',
                                         file: artifactPath,
                                         type: pom.packaging],
                                        // Lets upload the pom.xml file for additional information for Transitive dependencies
                                        [artifactId: pom.artifactId,
                                         classifier: '',
                                         file: "pom.xml",
                                         type: "pom"]
                                ]
                        );
                    } else {
                        error "*** File: ${artifactPath}, could not be found";
                    }
                }
            }
        }
        stage('Build container') {
            steps {
                script {
                    pom = readMavenPom file: "pom.xml";
                    TAG = pom.version;
                    sh "docker build -t petclinic:${TAG} ."
                }
            }
        }
        stage('Deploy container') {
            steps {
                script {
                    pom = readMavenPom file: "pom.xml";
                    TAG = pom.version;
                    withCredentials([[$class: 'UsernamePasswordMultiBinding', credentialsId: 'nexus-jenkins-credentials', usernameVariable: 'DOCKERHUB_API_USERNAME', passwordVariable: 'DOCKERHUB_API_PASSWORD']]) {
                        sh "echo login to ${NEXUS_DOCKER_PRIVATE_REGISTRY}"
                        // sh "docker build -t petclinic:${TAG} ."
                        sh "docker login --username ${env.DOCKERHUB_API_USERNAME} --password ${env.DOCKERHUB_API_PASSWORD} ${NEXUS_DOCKER_PRIVATE_REGISTRY}"
                        sh "docker tag petclinic:${TAG} ${NEXUS_DOCKER_PRIVATE_REGISTRY}/petclinic:${TAG}"
                        sh "docker push ${NEXUS_DOCKER_PRIVATE_REGISTRY}/petclinic:${TAG}"
                        sh "docker logout ${NEXUS_DOCKER_PRIVATE_REGISTRY}"
                    }
                }
            }
        }
    }
}
