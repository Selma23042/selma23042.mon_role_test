pipeline {
    agent any

    environment {
        GIT_URL = "${env.GIT_URL}"
    }

    stages {
        stage('Test and Lint') {
            steps {
                script {
                    docker.image('python:3.10').inside('-v /var/run/docker.sock:/var/run/docker.sock -u root:root') {
                        sh '''
                            apt-get update && apt-get install -y apt-transport-https ca-certificates curl gnupg lsb-release
                            curl -fsSL https://download.docker.com/linux/debian/gpg | gpg --dearmor -o /usr/share/keyrings/docker-archive-keyring.gpg
                            echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] https://download.docker.com/linux/debian $(lsb_release -cs) stable" | tee /etc/apt/sources.list.d/docker.list > /dev/null
                            apt-get update && apt-get install -y docker-ce-cli

                            pip install --upgrade pip
                            pip install ansible ansible-lint molecule molecule-plugins[docker]

                            echo "🔍 Running ansible-lint..."
                            ansible-lint .

                            echo "🧪 Running molecule tests..."
                            molecule test
                        '''
                    }
                }
            }
        }

        stage('Build Role Archive') {
            steps {
                script {
                    docker.image('python:3.10').inside() {
                        sh '''
                            set -e

                            pip install --upgrade pip
                            pip install ansible

                            ROLE_NAME=$(grep -E "^[[:space:]]*role_name:" meta/main.yml | sed 's/.*role_name:[[:space:]]*//' | tr -d '"' || basename $(pwd))
                            ROLE_VERSION=$(grep -E "^[[:space:]]*version:" meta/main.yml | sed 's/.*version:[[:space:]]*//' | tr -d '"' | tr -d "'" || git describe --tags --always 2>/dev/null || echo "1.0.0")

                            ARCHIVE_NAME="${ROLE_NAME}-${ROLE_VERSION}.tar.gz"

                            echo "📦 Création de l'archive ${ARCHIVE_NAME}..."

                            tar --exclude='.git*' \
                                --exclude='molecule' \
                                --exclude='*.tar.gz' \
                                --exclude='__pycache__' \
                                --exclude='*.pyc' \
                                --exclude='Jenkinsfile' \
                                -czf "${ARCHIVE_NAME}" \
                                --transform="s,^,${ROLE_NAME}/," .

                            echo "📁 Archive créée : ${ARCHIVE_NAME}"
                            echo "ℹ️  Pour installer manuellement : ansible-galaxy role install ${ARCHIVE_NAME}"
                        '''
                    }
                }
            }

            post {
                always {
                    archiveArtifacts artifacts: '*.tar.gz', allowEmptyArchive: false
                }
                success {
                    echo '✅ Rôle archivé avec succès. Disponible pour installation locale.'
                }
                failure {
                    echo '❌ Échec de la création de l’archive.'
                }
            }
        }
    }
}

