pipeline {
    agent any

    tools {
        maven "MVN_HOME"
    }

    environment {
        // Nexus Configuration
        NEXUS_VERSION = "nexus3"
        NEXUS_PROTOCOL = "http"
        NEXUS_URL = "54.227.83.9:8081"
        NEXUS_REPOSITORY = "sabear_pipeline_deploy"
        NEXUS_CREDENTIAL_ID = "2408cef2-4f8b-4a72-957f-7f872d6af833"

        // SonarQube Configuration
        SONARQUBE_URL = "http://3.88.47.108:9000/"
        SONARQUBE_CREDENTIAL_ID = "SONARQUBE-TOKEN"

        // Tomcat Deployment Configuration
        TOMCAT_URL = "http://54.227.83.9:8080/manager/html"
        TOMCAT_CREDENTIAL_ID = "TOMCAT1"

        // Slack Notifications
        SLACK_CHANNEL = "#devops"
    }

    stages {
        stage('Clone Code') {
            steps {
                git branch: 'master', url: 'https://github.com/vsivakumarreddy/sabear_simplecutomerapp.git'
            }
        }

        stage('Build with Maven') {
            steps {
                sh 'mvn -Dmaven.test.failure.ignore=true clean install'
            }
        }

        stage('Run SonarQube Scan') {
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

        stage('Publish to Nexus') {
            steps {
                script {
                    def pom = readMavenPom file: 'pom.xml'
                    def warFileName = "${pom.artifactId}-${pom.version}.${pom.packaging}"
                    def artifactPath = "target/${warFileName}"

                    if (!fileExists(artifactPath)) {
                        error "Artifact not found: ${artifactPath}"
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
                            [artifactId: pom.artifactId, classifier: '', file: artifactPath, type: pom.packaging],
                            [artifactId: pom.artifactId, classifier: '', file: 'pom.xml', type: 'pom']
                        ]
                    )
                }
            }
        }

        stage('Deploy to Tomcat') {
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
                    contextPath: "/shiva-app", // Deploy as "shiva-app"
                    war: warFile
                }
            }
        }
    }

    post {
        success {
            slackSend(channel: "${SLACK_CHANNEL}", message: "✅ *SUCCESS*: `sabear_simplecutomerapp` deployed as `/shiva-app`. [View Build](${env.BUILD_URL})", color: "#36a64f")
        }
        failure {
            slackSend(channel: "${SLACK_CHANNEL}", message: "❌ *FAILURE*: `sabear_simplecutomerapp` failed to deploy as `/shiva-app`. [View Build](${env.BUILD_URL})", color: "#ff0000")
        }
    }
}
