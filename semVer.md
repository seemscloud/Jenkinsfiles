```groovy
pipeline {
    environment {
        dockerRegistry = "theanotherwise/semver"
        dockerRegistryCredential = 'dockerhub_theanotherwise'

        repositoryCredentials = 'github_theanotherwise'

        dockerImage = ''

        semver_new_tag = ''
        semver_latest_tag = 'latest'
    }
    agent {
        docker {
            image 'docker:20.10.8'
            reuseNode true
        }
    }
    stages {
        stage('Build SemVer'){
            stages {
                stage('Clone Repository') {
                    agent {
                        docker {
                            image 'alpine:3.13.6'
                            reuseNode true
                        }
                    }
                    steps {
                        git branch: 'main', credentialsId: repositoryCredentials, url: 'git@github.com:theanotherwise/semver-docker.git'
                    }
                }
                stage('Bump Version') {
                    agent {
                        docker {
                            image 'alpine:3.13.6'
                            reuseNode true
                        }
                    }
                    steps {
                        withCredentials([sshUserPrivateKey(credentialsId: 'github_theanotherwise', keyFileVariable: 'SSH_KEY')]) {
                            sh '''
                                apk add --update --no-cache openssh git
                                VERSION="`cat VERSION`"
                                echo "$((VERSION+1))" > VERSION
                                git config --global user.email "version.bump@seems.cloud"
                                git config --global user.name "Version Bump"
                                mkdir -p ~/.ssh
                                ssh-keyscan github.com >> ~/.ssh/known_hosts
                                git add .
                                git commit -m "VERSION bump -> ${VERSION}"
                                GIT_SSH_COMMAND="ssh -i $SSH_KEY" git push --set-upstream origin main
                            '''
                        }

                        script {
                            semver_new_tag = readFile "VERSION"
                        }

                        stash includes: '*', name: 'semver'
                    }
                }
                stage('Build Image') {
                    agent {
                        docker {
                            image 'docker:20.10.8'
                            reuseNode true
                        }
                    }

                    steps {
                        script {
                            unstash 'semver'

                            dockerImage = docker.build(dockerRegistry)
                        }
                    }
                }
                stage('Test Image') {
                    agent {
                        docker {
                            image 'docker:20.10.8'
                            reuseNode true
                        }
                    }
                    steps {
                        script {
                            dockerImage.inside {
                                sh 'pip3 list'
                            }
                        }
                    }
                }
                stage('Publish Image') {
                    agent {
                        docker {
                            image 'docker:20.10.8'
                            reuseNode true
                        }
                    }
                    steps {
                        script {
                            docker.withRegistry('', dockerRegistryCredential ) {
                                dockerImage.push(semver_new_tag)
                                dockerImage.push(semver_latest_tag)
                            }
                        }
                    }
                }

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
