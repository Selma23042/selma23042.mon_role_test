pipeline {
  agent any
  environment {
    GALAXY_TOKEN = credentials('galaxy_token')
  }
  stages {
    stage('Setup') {
      steps {
        script {
          docker.image('python:3.10').inside('-v /var/run/docker.sock:/var/run/docker.sock -u root:root') {
            sh '''
              # Install Docker CLI
              apt-get update
              apt-get install -y apt-transport-https ca-certificates curl gnupg lsb-release
              curl -fsSL https://download.docker.com/linux/debian/gpg | gpg --dearmor -o /usr/share/keyrings/docker-archive-keyring.gpg
              echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] https://download.docker.com/linux/debian $(lsb_release -cs) stable" | tee /etc/apt/sources.list.d/docker.list > /dev/null
              apt-get update
              apt-get install -y docker-ce-cli
              
              # Install Python dependencies
              pip install ansible ansible-lint molecule molecule-plugins docker
              
              # Run linting
              ansible-lint .
              
              # Run molecule tests
              molecule test
              
              # Build and publish to Galaxy
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
