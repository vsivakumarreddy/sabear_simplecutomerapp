pipeline {
    agent { label "master" }

    tools {
        maven "${params.MAVEN_TOOL}" // Pass tool name like "MVN_HOME" as parameter
    }

    environment {
        // Credentials (Hardcoded)
        NEXUS_CREDENTIAL_ID = "2408cef2-4f8b-4a72-957f-7f872d6af833"
        SONARQUBE_CREDENTIAL_ID = "SONARQUBE-TOKEN"
        TOMCAT_CREDENTIAL_ID = "TOMCAT1"

        // Parameters (Set these in Jenkins job configuration)
        NEXUS_VERSION    = "${params.NEXUS_VERSION}"
        NEXUS_PROTOCOL   = "${params.NEXUS_PROTOCOL}"
        NEXUS_URL        = "${params.NEXUS_URL}"
        NEXUS_REPOSITORY = "${params.NEXUS_REPOSITORY}"

        SONARQUBE_URL    = "${params.SONARQUBE_URL}"

        TOMCAT_URL       = "${params.TOMCAT_URL}"

        SLACK_CHANNEL    = "${params.SLACK_CHANNEL}"
    }

    parameters {
        string(name: 'MAVEN_TOOL', defaultValue: 'MVN_HOME', description: 'Maven tool name from Jenkins configuration')
        string(name: 'NEXUS_URL', defaultValue: '54.227.22.45:8081', description: 'Nexus URL')
        choice(name: 'NEXUS_VERSION', choices: ['nexus2', 'nexus3'], description: 'Nexus version')
        choice(name: 'NEXUS_PROTOCOL', choices: ['http', 'https'], description: 'Nexus protocol')
        string(name: 'NEXUS_REPOSITORY', defaultValue: 'sabear_pipeline_deploy', description: 'Nexus repository name')
        string(name: 'SONARQUBE_URL', defaultValue: 'http://52.23.206.60:9000/', description: 'SonarQube server URL')
        string(name: 'TOMCAT_URL', defaultValue: 'http://54.227.22.45:8080/manager/html', description: 'Tomcat Manager URL')
        string(name: 'SLACK_CHANNEL', defaultValue: '#devops', description: 'Slack channel')
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
                    contextPath: "/${pom.artifactId}",
                    war: warFile
                }
            }
        }
    }

    post {
        success {
            slackSend(channel: "${SLACK_CHANNEL}", message: "✅ *SUCCESS*: Build & Deploy for `sabear_simplecutomerapp` on Jenkins. See build: ${env.BUILD_URL}", color: "#36a64f")
        }
        failure {
            slackSend(channel: "${SLACK_CHANNEL}", message: "❌ *FAILURE*: Build & Deploy for `sabear_simplecutomerapp` on Jenkins. See build: ${env.BUILD_URL}", color: "#ff0000")
        }
    }
}
