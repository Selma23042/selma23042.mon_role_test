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
                            pip install --upgrade pip
                            pip install ansible ansible-lint molecule molecule-plugins[docker]

                            # Run linting
                            echo "🔍 Running ansible-lint..."
                            ansible-lint .

                            # Run molecule tests
                            echo "🧪 Running molecule tests..."
                            molecule test
                        '''
                    }
                }
            }
        }

        stage('Build and Publish to Galaxy') {
            steps {
                script {
                    docker.image('python:3.10').inside() {
                        sh '''
                            # Install ansible
                            pip install --upgrade pip
                            pip install ansible

                            # Build the role
                            echo "📦 Building the role..."
                            ansible-galaxy role build

                            # Publish the role using .tar.gz archive
                            echo "🚀 Publishing to Ansible Galaxy..."
                            ROLE_ARCHIVE=$(ls *.tar.gz | head -n1)
                            ansible-galaxy role publish "$ROLE_ARCHIVE" --token "$GALAXY_TOKEN"

                            if [ $? -eq 0 ]; then
                                echo "✅ Role successfully published to Galaxy!"
                            else
                                echo "❌ Publication failed. Check the logs above for details."
                                exit 1
                            fi
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
            echo '🎉 Pipeline completed successfully!'
            echo '📦 Your role has been published to Ansible Galaxy.'
            echo '📥 Install with: ansible-galaxy install Selma23042.mon_role_test'
        }
        failure {
            echo '❌ Pipeline failed! Check the logs for more details.'
        }
        unstable {
            echo '⚠️  Pipeline completed with warnings.'
        }
    }
}
