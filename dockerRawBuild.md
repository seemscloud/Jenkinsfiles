```groovy
def dockerBuild() {
    tags = dockerTags()
    
    tags.each { item ->
        sh """
            cd application
            docker build . -t ${item}
        """
    }
}

def dockerPush(creds) {
    dockerLogin(creds)
    tags = dockerTags()
    
    tags.each { item ->
        sh """
            docker push ${item}
        """
    }
}

def dockerLogin(creds) {
    withCredentials([string(credentialsId: creds, variable: 'DOCKERHUB_PASSWORD')]) {
        sh "echo ${DOCKERHUB_PASSWORD} | docker login -u ${DOCKERHUB_USER} --password-stdin"
    }
}

def dockerTags() {
    return ["${DOCKERHUB_USER}/${JOB_BASE_NAME}:${BUILD_NUMBER}", "${DOCKERHUB_USER}/${JOB_BASE_NAME}:latest"]
}

pipeline {
    agent any
    
    environment {
        DOCKERHUB_USER = 'theanotherwise'
    }
    
    options {
        timeout(time: 5, unit: 'MINUTES') 
    }
    stages {
        stage('Clone'){
            steps {
                sh "env"
                git branch: 'main', credentialsId: 'df6af479-0544-4b53-8659-0d1ffc194302', url: 'git@github.com:seemscloud-testdrive/urban-take-home-application.git'
            }
        }
        stage('Build'){
            steps {
                dockerBuild()
            }
        }
        stage('Push'){
            steps {
                dockerPush('02952749-e1bd-4b26-a0a9-5d706b98b87f')
            }
        }
    }
}
```
