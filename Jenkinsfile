pipeline {
    agent any

    tools {
        maven 'MVN_HOME'
    }

    environment {
        // Credentials configured in Jenkins
        NEXUS_CREDENTIAL_ID     = "2408cef2-4f8b-4a72-957f-7f872d6af833"
        SONARQUBE_CREDENTIAL_ID = "SONARQUBE-TOKEN"
        TOMCAT_CREDENTIAL_ID    = "TOMCAT1"

        // Other configurations
        NEXUS_VERSION    = "nexus3"
        NEXUS_PROTOCOL   = "http"
        NEXUS_URL        = "54.227.22.45:8081"
        NEXUS_REPOSITORY = "sabear_pipeline_deploy"
        SONARQUBE_URL    = "http://52.23.206.60:9000/"
        TOMCAT_URL       = "http://54.227.22.45:8080/manager/html"
        SLACK_CHANNEL    = "#devops"
    }

    stages {
        stage('Clone Code') {
            steps {
                git branch: 'master', url: 'https://github.com/vsivakumarreddy/sabear_simplecutomerapp.git'
            }
        }

        stage('Build with Maven') {
            when { expression { return params.RUN_MAVEN_BUILD } }
            steps {
                sh 'mvn -Dmaven.test.failure.ignore=true clean install'
            }
        }

        stage('SonarQube Analysis') {
            when { expression { return params.RUN_SONAR_ANALYSIS } }
            steps {
                withSonarQubeEnv('sonarqube') {
                    withCredentials([string(credentialsId: "${SONARQUBE_CREDENTIAL_ID}", variable: 'SONAR_TOKEN')]) {
                        sh """
                            mvn sonar:sonar \
                                -Dsonar.projectKey=sabear_simplecutomerapp \
                                -Dsonar.host.url=$SONARQUBE_URL \
                                -Dsonar.login=$SONAR_TOKEN
                        """
                    }
                }
            }
        }

        stage('Deploy to Nexus') {
            when { expression { return params.DEPLOY_TO_NEXUS } }
            steps {
                script {
                    def pom = readMavenPom file: 'pom.xml'
                    def warFile = "target/${pom.artifactId}-${pom.version}.${pom.packaging}"
                    if (!fileExists(warFile)) {
                        error "WAR file not found: ${warFile}"
                    }

                    nexusArtifactUploader(
                        nexusVersion: NEXUS_VERSION,
                        protocol: NEXUS_PROTOCOL,
                        nexusUrl: NEXUS_URL,
                        repository: NEXUS_REPOSITORY,
                        credentialsId: NEXUS_CREDENTIAL_ID,
                        groupId: pom.groupId,
                        version: pom.version,
                        artifacts: [
                            [artifactId: pom.artifactId, classifier: '', file: warFile, type: pom.packaging],
                            [artifactId: pom.artifactId, classifier: '', file: 'pom.xml', type: 'pom']
                        ]
                    )
                }
            }
        }

        stage('Deploy to Tomcat') {
            when { expression { return params.DEPLOY_TO_TOMCAT } }
            steps {
                script {
                    def pom = readMavenPom file: 'pom.xml'
                    def warFile = "target/${pom.artifactId}-${pom.version}.${pom.packaging}"

                    deploy adapters: [
                        tomcat9(
                            credentialsId: TOMCAT_CREDENTIAL_ID,
                            path: '',
                            url: TOMCAT_URL
                        )
                    ],
                    contextPath: "/${pom.artifactId}",
                    war: warFile
                }
            }
        }
    }

    post {
        success {
            slackSend(channel: SLACK_CHANNEL, message: "✅ *SUCCESS*: Build & Deploy for `sabear_simplecutomerapp` - ${env.BUILD_URL}", color: "#36a64f")
        }
        failure {
            slackSend(channel: SLACK_CHANNEL, message: "❌ *FAILURE*: Build & Deploy for `sabear_simplecutomerapp` - ${env.BUILD_URL}", color: "#ff0000")
        }
    }
}
