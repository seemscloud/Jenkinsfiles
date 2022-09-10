# Jenkinsfiles

```groove
pipeline {
    agent {
        docker { 
			image 'ubuntu:focal' 
        }
    }
    stages {
        stage('Install') {
            steps {
                sh 'apt-get update'
                sh 'apt-get install git bash -y'
            }
        }
        stage('Clone') {
            steps {
                git branch: 'main', credentialsId: 'xxxxx', url: 'xxxxx'
            }
        }
        stage('Configure Git') {
            steps {
                withCredentials([sshUserPrivateKey(credentialsId: 'xxxxx', keyFileVariable: 'SSH_KEY')]) {
                    withEnv(["GIT_SSH_COMMAND=ssh -o StrictHostKeyChecking=no -i $SSH_KEY"]) {
                        sh 'git branch -a'
                        sh 'git config --global user.email "semver@host.domain"'
                        sh 'git config --global user.name "Semantic Versioning"'
                        sh 'git push --set-upstream origin main'
                    }
                }
            }
        }
        stage('Semantic Versioning') {
            agent {
                docker { 
        			image 'python:3.9.7'
        			reuseNode true
                }
            }
            steps {
                sh 'python3 -m pip install --upgrade pip'
                sh 'pip3 install semver'
                sh 'python3 semver.py'
            }
        }
    }
    post {
        always {
            cleanWs()
        }
    }
}
```
