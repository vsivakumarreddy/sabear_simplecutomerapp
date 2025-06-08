pipeline {
    agent { label "master" }
    tools {
        maven "MVN_HOME"
    }
    environment {
        NEXUS_VERSION = "nexus3"
        NEXUS_PROTOCOL = "http"
        NEXUS_URL = "54.221.153.41:8081"
        NEXUS_REPOSITORY = "sabear_simplecutomerapp-feature-1.1"
        NEXUS_CREDENTIAL_ID = "2408cef2-4f8b-4a72-957f-7f872d6af833"
        
        SONARQUBE_URL = "http://100.24.25.6:9000"
        SONARQUBE_CREDENTIAL_ID = "SONARQUBE-TOKEN"
        
        TOMCAT_URL = "http://54.221.153.41:8080/manager/html"
        TOMCAT_CREDENTIAL_ID = "TOMCAT1"
    }
    stages {
        stage('Clone Code') {
            steps {
                git 'https://github.com/betawins/sabear_simplecutomerapp.git'
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
                    sh 'mvn sonar:sonar -Dsonar.projectKey=sabear_simplecutomerapp -Dsonar.host.url=$SONARQUBE_URL -Dsonar.login=$SONARQUBE_CREDENTIAL_ID'
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
                deploy adapters: [tomcat9(credentialsId: TOMCAT_CREDENTIAL_ID, path: '', url: TOMCAT_URL)], contextPath: '/', war: "target/*.${pom.packaging}"
            }
        }
    }
}
