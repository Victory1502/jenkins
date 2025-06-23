pipeline {
    agent any
    
    tools { 
        nodejs 'NodeJS-18'
    }
    
    options {
        timeout(time: 30, unit: 'MINUTES')
    }
    
    environment {
        SSH_GITHUB = 'ssh-github'
    }
    
    stages {
        stage('Checkout') {
            steps {
                git branch: 'master', credentialsId: SSH_GITHUB, url: 'git@github.com:Victory1502/jenkins.git'
            }
        }
        
        stage('Install Dependencies') {
            steps {
                sh 'npm ci'
            }
        }
        
        stage('Lint') {
            steps {
                sh 'npm run lint'
            }
        }
        
        stage('Build') {
            steps {
                sh 'npm run build'
            }
        }
    }
    
    post {
        always {
            cleanWs()
        }
        
        success { 
            echo "✅ Pipeline completed successfully!"
        }
        
        failure {
            echo "❌ Pipeline failed!"
        }
    }
}