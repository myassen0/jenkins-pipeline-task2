pipeline {
    agent any
    
    options {
        skipDefaultCheckout true
        timeout(time: 30, unit: 'MINUTES')
    }
    
    environment {
        REPO_URL = 'https://github.com/myassen0/jenkins-pipeline-task2.git'
        DOCKERHUB_CREDENTIALS = credentials('dockerhub-creds')
    }
    
    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }
        
        stage('Build') {
            steps {
                script {
                    // Create a simple file to simulate build
                    writeFile file: 'app.txt', text: "Built at ${new Date()}"
                    archiveArtifacts artifacts: 'app.txt', fingerprint: true
                }
            }
        }
        
        stage('Test') {
            steps {
                script {
                    // Simulate tests
                    echo "Running tests..."
                    sh 'echo "Tests passed" > test-results.txt'
                    junit 'test-results.txt'
                }
            }
        }
        
        stage('Docker Build') {
            steps {
                script {
                    // Create a simple Dockerfile
                    writeFile file: 'Dockerfile', text: """
                    FROM alpine:latest
                    COPY app.txt /app/
                    CMD cat /app/app.txt
                    """
                    
                    // Build and push (commented out as we don't have Docker Hub repo yet)
                    // docker.build("yourdockerhubusername/jenkins-demo:${env.BUILD_NUMBER}").push()
                }
            }
        }
    }
    
    post {
        always {
            cleanWs()
            script {
                currentBuild.description = "Build #${env.BUILD_NUMBER}"
            }
        }
    }
}