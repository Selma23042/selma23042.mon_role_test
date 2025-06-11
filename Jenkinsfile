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
                branch 'main'
                tag pattern: 'v*', comparator: 'REGEXP'
            }
            steps {
                script {
                    docker.image('python:3.10').inside() {
                        sh '''
                            # Install ansible
                            pip install ansible
                            
                            # Import du rôle vers Galaxy avec tous les paramètres nécessaires
                            echo "Importing role to Ansible Galaxy..."
                            
                            ansible-galaxy role import \\
                                --server https://galaxy.ansible.com \\
                                --token "$GALAXY_TOKEN" \\
                                --branch main \\
                                --role-name mon_role_test \\
                                Selma23042 selma23042.mon_role_test
                            
                            # Vérifier le statut de l'import
                            echo "Import command completed. Check Galaxy for status."
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
            echo 'Pipeline completed successfully! Check https://galaxy.ansible.com/Selma23042/mon_role_test'
        }
        failure {
            echo 'Pipeline failed!'
        }
    }
}
