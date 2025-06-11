pipeline {
    agent { label "master" }

    tools {
        maven "MVN_HOME"
    }

    stages {
        stage('Clone Code') {
            steps {
                git branch: "${params.GIT_BRANCH}", url: 'https://github.com/vsivakumarreddy/sabear_simplecutomerapp.git'
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
                    withCredentials([string(credentialsId: "${params.SONARQUBE_CREDENTIAL_ID}", variable: 'SONAR_TOKEN')]) {
                        sh """
                            mvn sonar:sonar \
                                -Dsonar.projectKey=sabear_simplecutomerapp \
                                -Dsonar.host.url=${params.SONARQUBE_URL} \
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
                        error "Artifact not found: ${artifactPath}. Check Maven build and packaging type."
                    }

                    nexusArtifactUploader(
                        nexusVersion: "${params.NEXUS_VERSION}",
                        protocol: "${params.NEXUS_PROTOCOL}",
                        nexusUrl: "${params.NEXUS_URL}",
                        repository: "${params.NEXUS_REPOSITORY}",
                        credentialsId: "${params.NEXUS_CREDENTIAL_ID}",
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
                            credentialsId: "${params.TOMCAT_CREDENTIAL_ID}",
                            path: '',
                            url: "${params.TOMCAT_URL}"
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
            slackSend(channel: "${params.SLACK_CHANNEL}", message: "✅ *SUCCESS*: Build & Deploy for `sabear_simplecutomerapp`. See build: ${env.BUILD_URL}", color: "#36a64f")
        }
        failure {
            slackSend(channel: "${params.SLACK_CHANNEL}", message: "❌ *FAILURE*: Build & Deploy for `sabear_simplecutomerapp`. See build: ${env.BUILD_URL}", color: "#ff0000")
        }
    }
}
