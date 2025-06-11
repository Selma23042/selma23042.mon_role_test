pipeline {
  agent any
  environment {
    GALAXY_TOKEN = credentials('galaxy_token')
  }
  stages {
    stage('Setup') {
      steps {
        script {
          docker.image('python:3.10').inside('-v /var/run/docker.sock:/var/run/docker.sock') {
            sh '''
              pip install ansible ansible-lint molecule molecule-plugins
              ansible-lint .
              molecule test
              ansible-galaxy login --token $GALAXY_TOKEN
              ansible-galaxy role build
              ansible-galaxy role publish mon_role_test-*.tar.gz
            '''
          }
        }
      }
    }
  }
}
