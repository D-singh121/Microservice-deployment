pipeline {
    agent any

    stages {
        stage('Build & Tag Docker Image') {
            steps {
                script {
                    withDockerRegistry(credentialsId: 'docker-cred', toolName: 'docker') {
                        sh "docker build -t devesh121/adservice:latest ."
                    }
                }
            }
        }
        
        stage('Push Docker Image on Docker Hub') {
            steps {
                script {
                    withDockerRegistry(credentialsId: 'docker-cred', toolName: 'docker') {
                        sh "docker push devesh121/adservice:latest "
                    }
                }
            }
        }
    }
}
