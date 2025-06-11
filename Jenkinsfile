node('master') {
    // Define tools
    def mvnHome = tool name: 'MVN_HOME', type: 'hudson.tasks.Maven$MavenInstallation'

    // Define environment variables
    def NEXUS_VERSION = "nexus3"
    def NEXUS_PROTOCOL = "http"
    def NEXUS_URL = "54.227.22.45:8081"
    def NEXUS_REPOSITORY = "sabear_pipeline_deploy"
    def NEXUS_CREDENTIAL_ID = "2408cef2-4f8b-4a72-957f-7f872d6af833"

    def SONARQUBE_URL = "http://52.23.206.60:9000/"
    def SONARQUBE_CREDENTIAL_ID = "SONARQUBE-TOKEN"

    def TOMCAT_URL = "http://54.227.22.45:8080/manager/html"
    def TOMCAT_CREDENTIAL_ID = "TOMCAT1"

    def SLACK_CHANNEL = "#devops"

    try {
        stage('Clone Code') {
            git branch: 'master', url: 'https://github.com/vsivakumarreddy/sabear_simplecutomerapp.git'
        }

        stage('Build with Maven') {
            sh "${mvnHome}/bin/mvn -Dmaven.test.failure.ignore=true clean install"
        }

        stage('Run SonarQube Scan') {
            withSonarQubeEnv('sonarqube') {
                withCredentials([string(credentialsId: SONARQUBE_CREDENTIAL_ID, variable: 'SONAR_TOKEN')]) {
                    sh """
                        ${mvnHome}/bin/mvn sonar:sonar \
                            -Dsonar.projectKey=sabear_simplecutomerapp \
                            -Dsonar.host.url=${SONARQUBE_URL} \
                            -Dsonar.login=${SONAR_TOKEN}
                    """
                }
            }
        }

        stage('Publish to Nexus') {
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

        stage('Deploy to Tomcat') {
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

        // Success Slack Notification
        slackSend(channel: SLACK_CHANNEL, message: "✅ *SUCCESS*: Build & Deploy for `sabear_simplecutomerapp` on Jenkins. See build: ${env.BUILD_URL}", color: "#36a64f")

    } catch (Exception err) {
        // Failure Slack Notification
        slackSend(channel: SLACK_CHANNEL, message: "❌ *FAILURE*: Build & Deploy for `sabear_simplecutomerapp` on Jenkins. See build: ${env.BUILD_URL}", color: "#ff0000")
        throw err
    } finally {
        // Optional: clean workspace to free disk
        // deleteDir()
    }
}
