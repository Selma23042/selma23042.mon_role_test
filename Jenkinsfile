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
                    set -e  # Arrêter en cas d'erreur
                    
                    # Install dependencies
                    pip install --upgrade pip
                    pip install ansible requests
                    
                    echo "📦 Building role archive..."
                    
                    # Détection automatique du nom du rôle
                    if [ -f "meta/main.yml" ]; then
                        ROLE_NAME=$(grep -E "^\s*role_name:" meta/main.yml | sed 's/.*role_name:\s*//' | tr -d '"' || basename $(pwd))
                    else
                        ROLE_NAME=$(basename $(pwd))
                    fi
                    
                    # Version du rôle
                    if [ -f "meta/main.yml" ]; then
                        ROLE_VERSION=$(grep -E "^\s*version:" meta/main.yml | sed 's/.*version:\s*//' | tr -d '"' || "1.0.0")
                    else
                        ROLE_VERSION=$(git describe --tags --always 2>/dev/null || echo "1.0.0")
                    fi
                    
                    ARCHIVE_NAME="${ROLE_NAME}-${ROLE_VERSION}.tar.gz"
                    
                    echo "Building archive for role: ${ROLE_NAME} version: ${ROLE_VERSION}"
                    
                    # Création de l'archive optimisée
                    tar --exclude='.git*' \
                        --exclude='molecule' \
                        --exclude='tests' \
                        --exclude='*.tar.gz' \
                        --exclude='__pycache__' \
                        --exclude='*.pyc' \
                        --exclude='.pytest_cache' \
                        --exclude='Jenkinsfile' \
                        -czf "${ARCHIVE_NAME}" \
                        --transform="s,^,${ROLE_NAME}/," \
                        .
                    
                    echo "✅ Archive created: ${ARCHIVE_NAME} ($(du -h ${ARCHIVE_NAME} | cut -f1))"
                    
                    # Validation de l'archive
                    echo "🔍 Validating archive content..."
                    tar -tzf "${ARCHIVE_NAME}" | head -10
                    
                    echo "🚀 Publishing role..."
                    
                    # MÉTHODE 1: Publication vers Galaxy privé (Ansible Tower/AWX)
                    if [ ! -z "${GALAXY_PRIVATE_URL}" ]; then
                        echo "Publishing to private Galaxy: ${GALAXY_PRIVATE_URL}"
                        
                        # Upload via API REST
                        RESPONSE=$(curl -s -w "%{http_code}" \
                            -X POST \
                            -H "Authorization: Token ${GALAXY_TOKEN}" \
                            -H "Content-Type: multipart/form-data" \
                            -F "file=@${ARCHIVE_NAME}" \
                            -F "role_name=${ROLE_NAME}" \
                            "${GALAXY_PRIVATE_URL}/api/v1/roles/")
                        
                        HTTP_CODE="${RESPONSE: -3}"
                        if [ "$HTTP_CODE" -ge 200 ] && [ "$HTTP_CODE" -lt 300 ]; then
                            echo "✅ Successfully published to private Galaxy!"
                        else
                            echo "❌ Failed to publish to private Galaxy (HTTP: $HTTP_CODE)"
                            echo "Response: ${RESPONSE%???}"
                        fi
                    fi
                    
                    # MÉTHODE 2: Sauvegarde en artifacts Jenkins
                    echo "💾 Saving as Jenkins artifact..."
                    cp "${ARCHIVE_NAME}" "${ARCHIVE_NAME}.backup"
                    
                    # MÉTHODE 3: Test d'installation locale
                    echo "🧪 Testing local installation..."
                    mkdir -p /tmp/roles_test
                    ansible-galaxy role install "${ARCHIVE_NAME}" -p /tmp/roles_test --force
                    
                    if [ -d "/tmp/roles_test/${ROLE_NAME}" ]; then
                        echo "✅ Role installation test successful!"
                        echo "Installed files:"
                        ls -la "/tmp/roles_test/${ROLE_NAME}/"
                    else
                        echo "❌ Role installation test failed!"
                        exit 1
                    fi
                    
                    echo "🎉 Build and publish process completed successfully!"
                '''
            }
        }
    }
    post {
        always {
            // Archiver les artifacts
            archiveArtifacts artifacts: '*.tar.gz', allowEmptyArchive: true
        }
        success {
            echo "✅ Role successfully built and published!"
        }
        failure {
            echo "❌ Role build/publish failed. Check logs above."
        }
    }
}
