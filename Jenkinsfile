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

        stage('Publish to Galaxy (fallback if import fails)') {
            steps {
                script {
                    docker.image('python:3.10').inside() {
                        sh '''
                            set -e

                            pip install --upgrade pip
                            pip install ansible requests

                            echo "📦 Détection des infos du rôle..."

                            ROLE_NAME=$(grep -E "^[[:space:]]*role_name:" meta/main.yml | sed 's/.*role_name:[[:space:]]*//' | tr -d '"' || basename $(pwd))
                            ROLE_VERSION=$(grep -E "^[[:space:]]*version:" meta/main.yml | sed 's/.*version:[[:space:]]*//' | tr -d '"' | tr -d "'" || git describe --tags --always 2>/dev/null || echo "1.0.0")

                            ARCHIVE_NAME="${ROLE_NAME}-${ROLE_VERSION}.tar.gz"

                            GITHUB_USER=$(echo ${GIT_URL} | sed 's|https://github.com/||' | cut -d'/' -f1)
                            REPO_NAME=$(echo ${GIT_URL} | sed 's|https://github.com/.*/||' | sed 's|.git||')

                            echo "📌 GitHub: ${GITHUB_USER}/${REPO_NAME}"

                            echo "⚙️ Tentative de publication via IMPORT..."
                            if ansible-galaxy role import --token="${GALAXY_TOKEN}" "${GITHUB_USER}" "${REPO_NAME}"; then
                                echo "✅ IMPORT réussi sur Galaxy !"
                            else
                                echo "❌ IMPORT échoué. Fallback vers publication via archive..."

                                echo "📦 Construction de l'archive..."
                                tar --exclude='.git*' \
                                    --exclude='molecule' \
                                    --exclude='*.tar.gz' \
                                    --exclude='__pycache__' \
                                    --exclude='*.pyc' \
                                    --exclude='Jenkinsfile' \
                                    -czf "${ARCHIVE_NAME}" \
                                    --transform="s,^,${ROLE_NAME}/," .

                                echo "📤 Publication via .tar.gz..."
                                ansible-galaxy role publish "${ARCHIVE_NAME}" --token "${GALAXY_TOKEN}" || exit 1
                            fi
                        '''
                    }
                }
            }

            post {
                always {
                    archiveArtifacts artifacts: '*.tar.gz', allowEmptyArchive: true
                }
                success {
                    echo '🎉 Rôle publié avec succès sur Galaxy !'
                }
                failure {
                    echo '❌ Échec de la publication sur Ansible Galaxy.'
                }
            }
        }
    }
}
