pipeline {
  agent {
    docker {
      image 'python:3.10'
      args '-v /var/run/docker.sock:/var/run/docker.sock'
    }
  }
  environment {
    GALAXY_TOKEN = credentials('galaxy_token') // à créer dans Jenkins
  }
  stages {
    stage('Install tools') {
      steps {
        sh 'pip install ansible ansible-lint molecule[docker]'
      }
    }
    stage('Lint') {
      steps {
        sh 'ansible-lint .'
      }
    }
    stage('Molecule Test') {
      steps {
        sh 'molecule test'
      }
    }
    stage('Publish') {
      steps {
        sh 'ansible-galaxy login --token $GALAXY_TOKEN'
        sh 'ansible-galaxy role build'
        sh 'ansible-galaxy role publish mon_role_test-*.tar.gz'
      }
    }
  }
}
