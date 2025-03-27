pipeline {
    agent any
    tools {
        maven 'maven-3'
    }
    environment {
        // -----------------------------
        // Adjust these values as needed
        // -----------------------------
        JFROG_URL = "https://trialt0zppb.jfrog.io"
        JFROG_REPO_RELEASES = "petclinic-maven-dev-local"
        JFROG_REPO_SNAPSHOTS = "petclinic-maven-dev-virtual"
        JFROG_CREDENTIALS_ID = 'jfrog-saas'
        JFROG_CLI_BUILD_NAME = "spring-petclinic"
        JFROG_CLI_BUILD_NUMBER = "${BUILD_ID}"
        JF = "${WORKSPACE}/jfrog"  // local path to the downloaded CLI
    }
    stages {
        stage('Download JFrog CLI') {
            steps {
                sh """
                    curl -fL https://releases.jfrog.io/artifactory/jfrog-cli/v2-jf/2.74.1/jfrog-cli-linux-amd64/jf -o "${JF}"
                    chmod +x "${JF}"
                """
            }
        }

        stage('Configure JFrog CLI') {
            steps {
                withCredentials([usernamePassword(credentialsId: "${JFROG_CREDENTIALS_ID}", usernameVariable: 'JFROG_USER', passwordVariable: 'JFROG_API_KEY')]) {
                    sh """
                        ${JF} config add jenkins-config \
                            --artifactory-url=${JFROG_URL}/artifactory \
                            --user=${JFROG_USER} \
                            --password=${JFROG_API_KEY} \
                            --interactive=false \
                            --overwrite=true
                    """
                }
            }
        }

        stage('Build with Maven') {
            steps {
                sh """
                    ${JF} mvnc --global \\
                        --repo-resolve-releases=${JFROG_REPO_SNAPSHOTS} \\
                        --repo-resolve-snapshots=${JFROG_REPO_SNAPSHOTS} \\
                        --repo-deploy-releases=${JFROG_REPO_RELEASES} \\
                        --repo-deploy-snapshots=${JFROG_REPO_RELEASES}
                """
                sh """
                    ${JF} mvn clean deploy -DskipTests -Dcheckstyle.skip=true \\
                        --build-name=${JFROG_CLI_BUILD_NAME} \\
                        --build-number=${JFROG_CLI_BUILD_NUMBER}
                """
            }
        }

        stage('Xray Scan') {
            steps {
                // Scan the build you just deployed using Xray
                // Fail the build if there's a severity of "High" or above
                sh """
                    ${JF} xr build-scan ${JFROG_CLI_BUILD_NAME} ${JFROG_CLI_BUILD_NUMBER} \
                        --fail-on-severity=High
                """
            }
        }

        stage('Publish Build Info') {
            steps {
                sh """
                    ${JF} rt build-collect-env
                    ${JF} rt build-add-git
                    ${JF} rt build-publish
                """
            }
        }
    }
    post {
        always {
            echo "Build complete: ${env.JFROG_CLI_BUILD_NAME} #${env.BUILD_NUMBER}"
        }
    }
}
