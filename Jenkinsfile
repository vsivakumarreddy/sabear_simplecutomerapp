pipeline {
    agent { label "master" }

    tools {
        maven "Maven" // Ensure this matches your Maven tool name in Jenkins
    }

    environment {
        NEXUS_VERSION = "nexus3"
        NEXUS_PROTOCOL = "http"
        NEXUS_URL = "54.221.153.41:8081"
        NEXUS_REPOSITORY = "sabear_simplecutomerapp-feature-1.1"
        NEXUS_CREDENTIAL_ID = "2408cef2-4f8b-4a72-957f-7f872d6af833"

        SONARQUBE_URL = "http://100.24.25.6:9000"
        SONARQUBE_CREDENTIAL_ID = "SONARQUBE-TOKEN"

        TOMCAT_URL = "http://54.221.153.41:8080/manager/text"
        TOMCAT_CREDENTIAL_ID = "TOMCAT1"

        SLACK_CHANNEL = "#devops"
    }

    stages {
        stage('Clone Code') {
            steps {
                git branch: 'feature-1.1', url: 'https://github.com/vsivakumarreddy/sabear_simplecutomerapp.git'
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
                    def artifactPath = findFiles(glob: "target/*.${pom.packaging}")[0].path
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
                        tomcat9(credentialsId: TOMCAT_CREDENTIAL_ID, path: '', url: TOMCAT_URL)
                    ], contextPath: '/', war: warFile
                }
            }
        }
    }

    post {
        success {
            slackSend(channel: "${SLACK_CHANNEL}", message: "✅ Build & Deploy Successful for sabear_simplecutomerapp on Jenkins [jenkins-2nr9163]", color: "#36a64f")
        }
        failure {
            slackSend(channel: "${SLACK_CHANNEL}", message: "❌ Build Failed for sabear_simplecutomerapp on Jenkins [jenkins-2nr9163]", color: "#ff0000")
        }
    }
}
