pipeline {
  agent any
  environment {
    GALAXY_TOKEN = credentials('galaxy_token')
  }
  stages {
    stage('Test and Lint') {
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
            '''
          }
        }
      }
    }
    
    stage('Publish to Galaxy') {
      when {
        branch 'main'  // Only publish from main branch
      }
      steps {
        script {
          docker.image('python:3.10').inside() {
            sh '''
              # Install ansible
              pip install ansible
              
              # Method 1: Import directly from GitHub (Recommended)
              # This assumes your role is already pushed to GitHub
              ansible-galaxy role import --token="$GALAXY_TOKEN" Selma23042 mon_role_test
              
              # Alternative Method 2: Use Galaxy API directly with curl
              # Uncomment the following if you need to upload a tarball
              
              # # Create a proper tarball
              # tar --exclude='.git' --exclude='.pytest_cache' --exclude='__pycache__' \\
              #     --exclude='molecule/.cache' --exclude='*.pyc' \\
              #     -czf role.tar.gz .
              # 
              # # Upload using Galaxy API
              # RESPONSE=$(curl -s -X POST \\
              #   -H "Authorization: Token $GALAXY_TOKEN" \\
              #   -F "file=@role.tar.gz" \\
              #   https://galaxy.ansible.com/api/v1/imports/)
              # 
              # echo "Galaxy API Response: $RESPONSE"
              # 
              # # Check if import was successful
              # if echo "$RESPONSE" | grep -q '"state":"SUCCESS"\\|"state":"PENDING"'; then
              #   echo "Role import initiated successfully"
              # else
              #   echo "Role import failed"
              #   echo "$RESPONSE"
              #   exit 1
              # fi
            '''
          }
        }
      }
    }
  }
  
  post {
    always {
      cleanWs()
    }
    success {
      echo 'Pipeline completed successfully!'
    }
    failure {
      echo 'Pipeline failed!'
    }
  }
}
