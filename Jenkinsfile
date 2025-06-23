pipeline {
    agent any
    
    tools { 
        nodejs 'NodeJS-18' 
    }
    
    options {
        timeout(time: 60, unit: 'MINUTES')
        disableConcurrentBuilds()
    }
    
    environment {
        SSH_GITHUB = 'ssh-github'
        SSH_DEPLOY = 'ssh-deploy'
        REGISTRY_URL = 'myregistry/my-next-app'
        IMAGE_TAG = "${env.BUILD_NUMBER}"
        PREV_TAG = "${env.BUILD_NUMBER.toInteger() > 1 ? env.BUILD_NUMBER.toInteger() - 1 : 'latest'}"
    }
    
    stages {
        stage('Checkout') {
            steps {
                git branch: 'main', credentialsId: SSH_GITHUB, url: 'git@github.com:...'
            }
        }
        
        stage('Quality Gates') {
            parallel {
                stage('Unit Tests') {
                    steps { 
                        sh 'npm test -- --ci --coverage' 
                    }
                    post { 
                        always {
                            publishHTML(target: [
                                reportDir: 'coverage', 
                                reportFiles: 'index.html', 
                                reportName: 'Coverage'
                            ])
                        }
                    }
                }
                
                stage('Integration Tests') { 
                    steps { 
                        sh 'npm run test:integration' 
                    } 
                }
                
                stage('E2E Tests') {
                    when { branch 'main' }
                    steps { 
                        sh 'npm run test:e2e' 
                    }
                }
                
                stage('Lint & Security Audit') {
                    steps {
                        sh 'npm run lint -- --format=checkstyle --output-file=lint.xml'
                        recordIssues tools: [checkStyle(pattern: 'lint.xml')]
                        sh 'npm audit --audit-level high --json > audit.json'
                        archiveArtifacts artifacts: 'audit.json'
                    }
                }
            }
            post {
                always { 
                    publishTestResults testResultsPattern: '**/test-*.xml' 
                }
            }
        }
        
        stage('Build Docker') {
            options { 
                retry(2)
                timeout(time: 10, unit: 'MINUTES') 
            }
            steps { 
                sh 'docker build -t $REGISTRY_URL:$IMAGE_TAG -f Dockerfile.prod .' 
            }
        }
        
        // Sécurité: scan CVE
        stage('Scan Image') {
            steps { 
                sh 'trivy image --exit-code 1 --severity HIGH,CRITICAL $REGISTRY_URL:$IMAGE_TAG' 
            }
        }
        
        // Audit de devops
        stage('Docker Bench') {
            steps { 
                sh 'docker run --rm --net host --pid host docker/docker-bench-security' 
            }
        }
        
        stage('Push to Registry') {
            when { branch 'main' }
            steps {
                withCredentials([usernamePassword(
                    credentialsId: 'docker-registry',
                    usernameVariable: 'DOCKER_USER',
                    passwordVariable: 'DOCKER_PASS'
                )]) {
                    sh 'echo $DOCKER_PASS | docker login --username $DOCKER_USER --password-stdin myregistry'
                    sh 'docker push $REGISTRY_URL:$IMAGE_TAG'
                }
            }
        }
        
        stage('Pre-Deploy Validation') {
            steps {
                script {
                    sh "docker manifest inspect $REGISTRY_URL:$IMAGE_TAG"
                    sh "ssh -o ConnectTimeout=10 deploy@myserver 'echo OK -> server accessible'"
                    sh """
                        ssh deploy@myserver '
                        usage=\$(df / | tail -1 | awk "{print \\\$5}" | sed "s/%//")
                        if ((usage>90)); then echo "❌ Low disk"; exit 1; fi'
                    """
                }
            }
        }
        
        stage('Deploy') {
            when { branch 'main' }
            steps {
                sshagent([SSH_DEPLOY]) {
                    sh """
                        ssh deploy@myserver '
                        docker pull $REGISTRY_URL:$IMAGE_TAG &&
                        docker stop next-app || true &&
                        docker rm next-app || true &&
                        docker run -d --name next-app -p 80:3000 $REGISTRY_URL:$IMAGE_TAG'
                    """
                }
            }
        }
        
        stage('Health Check') {
            steps {
                script {
                    int max = 5, delay = 30
                    for(int i = 0; i < max; i++) {
                        try {
                            sh "curl --silent --fail --max-time 10 http://myserver/health"
                            echo "✅ Health OK"
                            break
                        } catch (e) {
                            if(i == max-1) error "❌ Health failed after $max attempts"
                            echo "🔄 Retry ${i+1}/${max}"
                            sleep delay
                        }
                    }
                }
            }
        }
    }
    
    post {
        failure {
            script {
                try {
                    sshagent([SSH_DEPLOY]) {
                        sh """
                            ssh deploy@myserver '
                            docker pull $REGISTRY_URL:$PREV_TAG &&
                            docker stop next-app && docker rm next-app &&
                            docker run -d --name next-app -p 80:3000 $REGISTRY_URL:$PREV_TAG'
                        """
                    }
                } catch (Exception e) {
                    echo "⚠️ Rollback failed: ${e.message}"
                }
            }
        }
        
        always {
            cleanWs()
        }
        
        success { 
            slackSend(
                channel: '#deployments', 
                color: 'good',
                message: "✅ $JOB_NAME #$BUILD_NUMBER deployed!"
            )
        }
        
        failure {
            slackSend(
                channel: '#alerts', 
                color: 'danger',
                message: "❌ $JOB_NAME #$BUILD_NUMBER failed at stage ${env.STAGE_NAME}. Rollback triggered."
            )
        }
    }
}