```groovy
pipeline {
    agent {
        docker {
            image 'docker:20.10.8'
        }
    }
    stages {
        stage('Create'){
            parallel {
                stage('First') {
                    agent {
                        docker {
                            image 'alpine:3.13.6'
                        }
                    }
                    steps {
                        sh 'touch first'
                    }
                }
                stage('Second') {
                    agent {
                        docker {
                            image 'alpine:3.13.5'
                        }
                    }
                    steps {
                        sh 'touch second'
                    }
                }
                stage('Third') {
                    agent {
                        docker {
                            image 'alpine:3.13.4'
                        }
                    } 
                    steps {
                        sh 'touch third'
                    }
                }
            }   
        }
        stage('Summary') {
            agent {
                docker {
                    image 'alpine:3.13.3'
                }
            }
            steps {
                sh 'ls -lh'
            }
        }
    }
}
```
