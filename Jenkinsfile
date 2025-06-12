pipeline {
    agent any

    environment {
        // Clé API pour Ansible Galaxy privé
        ANSIBLE_GALAXY_API_KEY = credentials('ansible-galaxy-api-key')
        // URL de votre serveur Ansible Galaxy privé
        ANSIBLE_GALAXY_URL = 'https://your-private-galaxy.com/'
    }

    stages {
        // Étape pour tester et exécuter des vérifications lint avec ansible-lint et moleculer
        stage('Test and Lint') {
            steps {
                script {
                    // Utilisation d'un conteneur Docker Python avec accès Docker-in-Docker
                    docker.image('python:3.10').inside('-v /var/run/docker.sock:/var/run/docker.sock -u root:root') {
                        sh '''
                            echo "🔄 Mise à jour des paquets et installation des dépendances Docker..."

                            # Mise à jour des paquets et installation de Docker
                            apt-get update && apt-get install -y apt-transport-https ca-certificates curl gnupg lsb-release
                            curl -fsSL https://download.docker.com/linux/debian/gpg | gpg --dearmor -o /usr/share/keyrings/docker-archive-keyring.gpg
                            echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] https://download.docker.com/linux/debian $(lsb_release -cs) stable" | tee /etc/apt/sources.list.d/docker.list > /dev/null
                            apt-get update && apt-get install -y docker-ce-cli

                            # Installation des outils Python nécessaires
                            pip install --upgrade pip
                            pip install ansible ansible-lint molecule molecule-plugins[docker]

                            echo "🔍 Exécution de ansible-lint..."
                            ansible-lint .

                            echo "🧪 Exécution des tests Molecule..."
                            molecule test
                        '''
                    }
                }
            }
        }

        // Étape pour créer l'archive du rôle
        stage('Build Role Archive') {
            steps {
                script {
                    // Utilisation d'un conteneur Docker pour créer l'archive
                    docker.image('python:3.10').inside() {
                        sh '''
                            set -e

                            # Mise à jour et installation d'Ansible
                            pip install --upgrade pip
                            pip install ansible

                            # Extraction du nom du rôle et de sa version
                            ROLE_NAME=$(grep -E "^[[:space:]]*role_name:" meta/main.yml | sed 's/.*role_name:[[:space:]]*//' | tr -d '"' || basename $(pwd))
                            ROLE_VERSION=$(grep -E "^[[:space:]]*version:" meta/main.yml | sed 's/.*version:[[:space:]]*//' | tr -d '"' | tr -d "'" || git describe --tags --always 2>/dev/null || echo "1.0.0")

                            # Création du nom de l'archive
                            ARCHIVE_NAME="${ROLE_NAME}-${ROLE_VERSION}.tar.gz"

                            echo "📦 Création de l'archive ${ARCHIVE_NAME}..."

                            # Création de l'archive en excluant certains fichiers/dossiers
                            tar --exclude='.git*' \
                                --exclude='molecule' \
                                --exclude='*.tar.gz' \
                                --exclude='__pycache__' \
                                --exclude='*.pyc' \
                                --exclude='Jenkinsfile' \
                                -czf "${ARCHIVE_NAME}" \
                                --transform="s,^,${ROLE_NAME}/," .

                            echo "📁 Archive créée : ${ARCHIVE_NAME}"
                            echo "ℹ️ Pour installer manuellement : ansible-galaxy role install ${ARCHIVE_NAME}"
                        '''
                    }
                }
            }

            post {
                always {
                    // Archivage de l'archive créée
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

        // Étape pour publier le rôle sur Ansible Galaxy privé
        stage('Publish to Ansible Galaxy') {
            steps {
                script {
                    echo 'Publication du rôle validé sur Ansible Galaxy privé...'

                    // Connexion à Ansible Galaxy avec la clé API et l'URL de Galaxy privé
                    sh '''
                    ansible-galaxy login --api-key $ANSIBLE_GALAXY_API_KEY --server $ANSIBLE_GALAXY_URL
                    ansible-galaxy role publish ./roles/my_role --force --server $ANSIBLE_GALAXY_URL
                    '''
                }
            }
        }
    }
}

