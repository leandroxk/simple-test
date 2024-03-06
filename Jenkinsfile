@Library(['release-libraries', 'matera-build-analyzer-clean-code', 'ci-pipeline-utilities', 'tekton-library']) _

pipeline {

    options {
        skipDefaultCheckout true
        buildDiscarder(logRotator(numToKeepStr: '5'))
        disableConcurrentBuilds()
    }

    agent {
        node {
            label env.TEKTON_LABEL_OHIO
            customWorkspace env.TEKTON_WS
        }
    }

    stages {
        stage('checkout') {
            steps {
                script {
                    git branch: "main",
                        credentialsId: "github",
                        url: "git@github.com:leandroxk/simple-test.git"
                }
            }
        }

        stage('build') {
            steps {
                script {
                    tekton withMaven: "clean package", image: "harbor-dev.matera.com/docker-hub/library/maven:3.9.6-eclipse-temurin-17"
                }
            }
        }
        
        stage('analysis') {
            when {
                expression { pipelineSteps().shouldPerformExtraValidation() }
            }
            steps {
                script {
                    try {
                        withAWS(credentials: 'builder.devops', region: 'us-east-1') {
                            analyze()
                                .checkstyle()
                                .spotbugs()
                                .dependencyTracker(team_email:"leandro.corbelo@matera.com", pattern: "target/bom.xml")
                                .checkProjectVulnerabilities()
                                .defectDojo()
                                .commentingAfterwards {
                                    tekton(
                                        withMaven: "verify -Panalysis -DskipTests -Ddependency-check-maven-plugin.autoUpdate=false",
                                        archiveFiles: "analysis=**/*.xml",
                                        resourcesTier: "high",
                                        image: TEKTON_IMAGE
                                    );
                            }
                        }
                    } catch (ex) {
                        unstable(message: "Error while trying to analyze the project.")
                    }
                }
            }
        }
    
        
    }
}