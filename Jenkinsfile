pipeline {
    agent any 
    
    stages { 
        stage('SCM Checkout') {
            steps {
                retry(3) {
                    git branch: 'main', url: 'https://github.com/kavishkaRash/github-docker-and-genkins-ci-cd-pipeline'
                }
            }
        }

        stage('Build Docker Image') {
            steps {  
                sh "docker build -t rushferz/my-nodejs-app:v1.0.${BUILD_NUMBER} ."
            }
        }

        stage('Login to Docker Hub') {
            steps {
                withCredentials([string(credentialsId: 'dockerHub-password', variable: 'dockerHubPass')]) {
                    sh "docker login -u rushferz -p ${dockerHubPass}"
                }
            }
        }

        stage('Push Image') {
            steps {
                sh "docker push rushferz/my-nodejs-app:v1.0.${BUILD_NUMBER}"
            }
        }
    }

    post {
        always {
            sh 'docker logout'
        }
    }
}