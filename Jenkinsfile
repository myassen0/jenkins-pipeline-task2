pipeline {
    agent any
    
    options {
        skipDefaultCheckout true
        timeout(time: 30, unit: 'MINUTES')
    }
    
    environment {
        REPO_URL = 'https://github.com/myassen0/jenkins-pipeline-task2.git'
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
                    writeFile file: 'app.txt', text: "Built at ${new Date()}"
                    archiveArtifacts artifacts: 'app.txt', fingerprint: true
                }
            }
        }
        
        stage('Test') {
            steps {
                script {
                    // Simulate test in JUnit XML format
                    writeFile file: 'test-results.xml', text: '''
                    <testsuite tests="1"><testcase classname="demo" name="test1"/></testsuite>
                    '''
                    junit 'test-results.xml'
                }
            }
        }
        
        stage('Docker Build and Push') {
            steps {
                script {
                    writeFile file: 'Dockerfile', text: """
                    FROM alpine:latest
                    COPY app.txt /app/
                    CMD cat /app/app.txt
                    """
                    
                    withCredentials([usernamePassword(credentialsId: 'docker-hub-creds', usernameVariable: 'DOCKER_USER', passwordVariable: 'DOCKER_PASS')]) {
                        sh "echo $DOCKER_PASS | docker login -u $DOCKER_USER --password-stdin"
                        sh "docker build -t $DOCKER_USER/jenkins-demo:${env.BUILD_NUMBER} ."
                        sh "docker push $DOCKER_USER/jenkins-demo:${env.BUILD_NUMBER}"
                    }
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
