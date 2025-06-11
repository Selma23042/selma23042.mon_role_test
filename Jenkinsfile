pipeline {
    agent any

    environment {
        GALAXY_TOKEN = credentials('galaxy_token')
        GIT_URL = "${env.GIT_URL}"
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
                            set -e  # Arrêter en cas d'erreur
                            
                            # Install dependencies
                            pip install --upgrade pip
                            pip install ansible requests
                            
                            echo "📦 Building role archive..."
                            
                            # Détection automatique du nom du rôle
                            if [ -f "meta/main.yml" ]; then
                                ROLE_NAME=$(grep -E "^[[:space:]]*role_name:" meta/main.yml | sed 's/.*role_name:[[:space:]]*//' | tr -d '"' || basename $(pwd))
                            else
                                ROLE_NAME=$(basename $(pwd))
                            fi
                            
                            # Version du rôle
                            if [ -f "meta/main.yml" ]; then
                                ROLE_VERSION=$(grep -E "^[[:space:]]*version:" meta/main.yml | sed 's/.*version:[[:space:]]*//' | tr -d '"' | tr -d "'" || echo "1.0.0")
                                if [ -z "$ROLE_VERSION" ]; then
                                    ROLE_VERSION=$(git describe --tags --always 2>/dev/null || echo "1.0.0")
                                fi
                            else
                                ROLE_VERSION=$(git describe --tags --always 2>/dev/null || echo "1.0.0")
                            fi
                            
                            ARCHIVE_NAME="${ROLE_NAME}-${ROLE_VERSION}.tar.gz"
                            
                            echo "Building archive for role: ${ROLE_NAME} version: ${ROLE_VERSION}"
                            
                            # Création de l'archive optimisée
                            tar --exclude='.git*' \\
                                --exclude='molecule' \\
                                --exclude='tests' \\
                                --exclude='*.tar.gz' \\
                                --exclude='__pycache__' \\
                                --exclude='*.pyc' \\
                                --exclude='.pytest_cache' \\
                                --exclude='Jenkinsfile' \\
                                --exclude='venv' \\
                                --exclude='env' \\
                                --exclude='.venv' \\
                                --exclude='node_modules' \\
                                --exclude='.DS_Store' \\
                                -czf "${ARCHIVE_NAME}" \\
                                --transform="s,^,${ROLE_NAME}/," \\
                                .
                            
                            echo "✅ Archive created: ${ARCHIVE_NAME} ($(du -h ${ARCHIVE_NAME} | cut -f1))"
                            
                            # Validation de l'archive
                            echo "🔍 Validating archive content..."
                            tar -tzf "${ARCHIVE_NAME}" | head -10
                            
                            echo "🚀 Publishing role to Ansible Galaxy..."
                            
                            # Publication vers Ansible Galaxy public
                            GITHUB_USER=$(echo ${GIT_URL} | sed 's|https://github.com/||' | cut -d'/' -f1)
                            REPO_NAME=$(echo ${GIT_URL} | sed 's|https://github.com/.*/||' | sed 's|.git||')
                            
                            echo "GitHub User: ${GITHUB_USER}"
                            echo "Repository: ${REPO_NAME}"
                            
                            # Vérifier que le namespace existe
                            echo "⚠️  IMPORTANT: Assurez-vous que le namespace '${GITHUB_USER}' existe sur Ansible Galaxy"
                            echo "   Visitez: https://galaxy.ansible.com/ui/standalone/namespaces/${GITHUB_USER}/"
                            
                            ansible-galaxy role import --token=${GALAXY_TOKEN} ${GITHUB_USER} ${REPO_NAME}
                            
                            if [ $? -eq 0 ]; then
                                echo "✅ Successfully published to Ansible Galaxy!"
                            else
                                echo "❌ Failed to publish to Ansible Galaxy"
                                echo "💡 Vérifiez que:"
                                echo "   - Le namespace '${GITHUB_USER}' existe sur Galaxy"
                                echo "   - Votre token Galaxy est valide"
                                echo "   - Le repository est public"
                                exit 1
                            fi
                            
                            echo "🎉 Build and publish process completed successfully!"
                        '''
                    }
                }
            }
            post {
                always {
                    // Archiver l'archive créée
                    archiveArtifacts artifacts: '*.tar.gz', allowEmptyArchive: true
                }
                success {
                    echo "✅ Role successfully built and published to Ansible Galaxy!"
                }
                failure {
                    echo "❌ Role build/publish failed. Check logs above."
                }
            }
        }
    }
}
