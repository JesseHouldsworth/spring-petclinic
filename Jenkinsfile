pipeline {
  agent any

  tools {
    // This is your Jenkins-managed Maven tool
    maven 'maven-3'
  }

  environment {
    JFROG_CLI_BUILD_NAME = "spring-petclinic"
    JFROG_CLI_BUILD_NUMBER = "${BUILD_ID}"
    ARTIFACTORY_URL = "http://artifactory.artifactory.svc.cluster.local:8081/artifactory"
  }

  stages {

    stage('Download JFrog CLI') {
      steps {
        // Download a *confirmed* ARM64 version – 2.44.0
        sh '''
          curl -fL https://releases.jfrog.io/artifactory/jfrog-cli/v2-jf/2.44.0/jfrog-cli-linux-arm64/jf -o jf
          chmod +x jf
        '''
      }
    }

    stage('Configure JFrog CLI') {
      steps {
        withCredentials([usernamePassword(credentialsId: 'jfrog-platform-creds', usernameVariable: 'ARTIFACTORY_USER', passwordVariable: 'ARTIFACTORY_PASSWORD')]) {
          sh '''
            ./jf c add petclinic \
              --url=${ARTIFACTORY_URL} \
              --user=$ARTIFACTORY_USER \
              --password=$ARTIFACTORY_PASSWORD \
              --interactive=false
          '''
        }
      }
    }

    stage('Validate Connection') {
      steps {
        // Switch to "petclinic" config & ping Artifactory
        sh './jf c use petclinic'
        sh './jf rt ping'
      }
    }

    stage('Build Maven') {
      steps {
        sh 'chmod +x mvnw'
        // Configure Maven resolution + deployment in Artifactory
        sh '''
          ./jf mvnc --global \
            --repo-resolve-releases=petclinic-maven-dev-virtual \
            --repo-resolve-snapshots=petclinic-maven-dev-virtual \
            --repo-deploy-releases=petclinic-maven-dev-local \
            --repo-deploy-snapshots=petclinic-maven-dev-local
        '''
        // Perform the actual build + deploy
        sh './jf mvn clean deploy -DskipTests -Dcheckstyle.skip=true'
      }
    }

    stage('Publish Build Info') {
      steps {
        sh './jf rt build-collect-env'
        sh './jf rt build-add-git'
        sh './jf rt build-publish'
      }
    }
  }

  post {
    always {
      echo "Build complete: ${env.JFROG_CLI_BUILD_NAME} #${env.BUILD_NUMBER}"
    }
  }
}
