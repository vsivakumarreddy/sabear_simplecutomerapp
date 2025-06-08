pipeline {
    agent {
        label "master"
    }
    tools {
        maven "MVN_HOME"               // Maven installation name in Jenkins
        sonarQubeScanner "sonarqube"   // SonarQube Scanner tool name in Jenkins
    }
    environment {
        // SonarQube Configuration
        SONAR_SCANNER_HOME = tool 'sonarqube'
        SONAR_QUBE_URL = "http://100.24.25.6:9000"     // Updated SonarQube URL

        // Nexus Configuration
        NEXUS_VERSION = "nexus3"
        NEXUS_PROTOCOL = "http"
        NEXUS_URL = "54.221.153.41:8081"               // Updated Nexus URL
        NEXUS_REPOSITORY = "soanrqube"
        NEXUS_CREDENTIAL_ID = "2408cef2-4f8b-4a72-957f-7f872d6af833"   // Updated Nexus Credential ID

        // Tomcat Deployment Configuration
        TOMCAT_URL = "http://54.221.153.41:8080/manager/text"  // Tomcat deployment API URL
        TOMCAT_CREDENTIAL_ID = "TOMCAT1"                       // Updated Tomcat Credential ID
    }

    stages {
        stage("clone code") {
            steps {
                script {
                    git 'https://github.com/betawins/sabear_simplecutomerapp.git'
                }
            }
        }

        stage("SonarQube Analysis") {
            steps {
                withSonarQubeEnv('SONAR1') { // SonarQube server name configured in Jenkins
                    sh "${SONAR_SCANNER_HOME}/bin/sonar-scanner"
                }
            }
        }

        stage("mvn build") {
            steps {
                sh 'mvn -Dmaven.test.failure.ignore=true install'
            }
        }

        stage("publish to nexus") {
            steps {
                script {
                    def pom = readMavenPom file: "pom.xml"
                    def filesByGlob = findFiles(glob: "target/*.${pom.packaging}")
                    def artifactPath = filesByGlob[0].path

                    if (fileExists(artifactPath)) {
                        echo "*** File: ${artifactPath}, group: ${pom.groupId}, packaging: ${pom.packaging}, version ${pom.version}"

                        nexusArtifactUploader(
                            nexusVersion: NEXUS_VERSION,
                            protocol: NEXUS_PROTOCOL,
                            nexusUrl: NEXUS_URL,
                            groupId: pom.groupId,
                            version: pom.version,
                            repository: NEXUS_REPOSITORY,
                            credentialsId: NEXUS_CREDENTIAL_ID,
                            artifacts: [
                                [artifactId: pom.artifactId,
                                classifier: '',
                                file: artifactPath,
                                type: pom.packaging],
                                [artifactId: pom.artifactId,
                                classifier: '',
                                file: "pom.xml",
                                type: "pom"]
                            ]
                        )
                    } else {
                        error "*** File: ${artifactPath}, could not be found"
                    }
                }
            }
        }

        stage("Deploy to Tomcat") {
            steps {
                script {
                    def warFile = findFiles(glob: 'target/*.war')[0]
                    def contextPath = "SimpleCustomerApp"

                    echo "Deploying ${warFile.name} to Tomcat..."

                    deploy adapters: [tomcat8(credentialsId: TOMCAT_CREDENTIAL_ID,
                                              path: "",
                                              url: TOMCAT_URL)],
                           contextPath: contextPath,
                           war: warFile.path
                }
            }
        }
    }
}
