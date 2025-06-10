pipeline {
    agent { label "master" } // Consider changing "master" to a dedicated agent label for production workloads

    tools {
        maven "MVN_HOME" // Ensure this matches your Maven tool name configured in Jenkins (Manage Jenkins -> Tools)
    }

    environment {
        // Nexus Configuration
        NEXUS_VERSION = "nexus3" // Or "nexus2" depending on your Nexus version
        NEXUS_PROTOCOL = "http"  // Keep as http unless your Nexus is configured for https
        NEXUS_URL = "18.234.108.178:8081" // CORRECTED: Removed the leading "http://" from here
        NEXUS_REPOSITORY = "sabear_pipeline_deploy"
        NEXUS_CREDENTIAL_ID = "2408cef2-4f8b-4a72-957f-7f872d6af833" // Ensure this ID is correct and active in Jenkins Credentials

        // SonarQube Configuration
        SONARQUBE_URL = "http://54.152.216.92:9000/" // Keep the full URL here
        SONARQUBE_CREDENTIAL_ID = "SONARQUBE-TOKEN" // Ensure this ID is correct and active in Jenkins Credentials

        // Tomcat Deployment Configuration
        TOMCAT_URL = "http://18.234.108.178:8080/manager/html" // URL to your Tomcat Manager App
        TOMCAT_CREDENTIAL_ID = "TOMCAT1" // Ensure this ID (Username/Password) is correct for Tomcat Manager

        // Slack Notifications
        SLACK_CHANNEL = "#devops" // Your Slack channel name
    }

    stages {
        stage('Clone Code') {
            steps {
                git branch: 'master', url: 'https://github.com/vsivakumarreddy/sabear_simplecutomerapp.git'
            }
        }

        stage('Build with Maven') {
            steps {
                // Using the Maven tool configured in Jenkins
                sh 'mvn -Dmaven.test.failure.ignore=true clean install'
                // -Dmaven.test.failure.ignore=true: Allows the build to proceed even if unit tests fail.
                // Consider removing this flag for strict quality gates in production.
            }
        }

        stage('Run SonarQube Scan') {
            steps {
                withSonarQubeEnv('sonarqube') { // 'sonarqube' should match the name of your SonarQube server configured in Jenkins
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
                    // Read Maven POM details
                    def pom = readMavenPom file: 'pom.xml' // Ensure pom.xml is at the root of the workspace
                    
                    // Construct the full path to the WAR file
                    // Assuming the WAR artifact is in target/ and follows Maven's default naming convention
                    def warFileName = "${pom.artifactId}-${pom.version}.${pom.packaging}"
                    def artifactPath = "target/${warFileName}"

                    // Verify the artifact exists before attempting to upload
                    if (!fileExists(artifactPath)) {
                        error "Artifact not found: ${artifactPath}. Check Maven build and packaging type."
                    }

                    // Upload artifacts to Nexus
                    nexusArtifactUploader(
                        nexusVersion: NEXUS_VERSION,
                        protocol: NEXUS_PROTOCOL,
                        nexusUrl: NEXUS_URL, // This will now correctly combine with NEXUS_PROTOCOL in the plugin
                        repository: NEXUS_REPOSITORY,
                        credentialsId: NEXUS_CREDENTIAL_ID,
                        groupId: pom.groupId,
                        version: pom.version,
                        artifacts: [
                            // Main artifact (WAR file)
                            [artifactId: pom.artifactId, classifier: '', file: artifactPath, type: pom.packaging],
                            // POM file for metadata
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
                    // Construct the full path to the WAR file for deployment
                    def warFile = "target/${pom.artifactId}-${pom.version}.${pom.packaging}"

                    // Deploy using the Tomcat plugin
                    deploy adapters: [
                        tomcat9(
                            credentialsId: TOMCAT_CREDENTIAL_ID,
                            path: '', // Optional: Tomcat manager path (usually empty for default)
                            url: TOMCAT_URL // URL to Tomcat Manager HTML interface
                        )
                    ],
                    // *******************************************************************
                    // CHANGED HERE: Dynamically set contextPath based on Maven artifactId
                    // *******************************************************************
                    contextPath: "/${pom.artifactId}", // The application context path on Tomcat (e.g., http://<tomcat-ip>:8080/SimpleCustomerApp)
                    war: warFile // The path to the WAR file to deploy
                }
            }
        }
    }

    // Post-build actions (notifications)
    post {
        success {
            slackSend(channel: "${SLACK_CHANNEL}", message: "✅ *SUCCESS*: Build & Deploy for `sabear_simplecutomerapp` on Jenkins. See build: ${env.BUILD_URL}", color: "#36a64f")
        }
        failure {
            slackSend(channel: "${SLACK_CHANNEL}", message: "❌ *FAILURE*: Build & Deploy for `sabear_simplecutomerapp` on Jenkins. See build: ${env.BUILD_URL}", color: "#ff0000")
        }
        // Always execute this block, regardless of success/failure
        // finally {
        //     // Optional: Clean up workspace after build to save disk space
        //     // deleteDir()
        // }
    }
}
