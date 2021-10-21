# Jenkinsfiles

```Jenkinsfile
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

```jenkinsfile
pipeline {
    agent {
        docker { 
			image 'alpine:3.13.5' 
        }
    }

    stages {
        stage('Prepare Environment') {
            steps {
                sh 'apk add unzip wget curl sed'
                sh 'wget -nv "https://releases.hashicorp.com/terraform/${TERRAFORM_VERSION}/terraform_${TERRAFORM_VERSION}_linux_amd64.zip" -O terraform.zip'
                sh 'unzip -o terraform.zip'
                sh 'mv -f terraform /usr/local/bin'
            }
        }
        
        stage('Clone Repository') {
            steps {
                git credentialsId: '537b3740-0f31-4e7a-9f80-a24227cb6a38', url: ""
            }
        }
        
        stage('Terraform Initialize'){
            steps {
                sh 'terraform init'
            }
        }
        
      	stage('Terraform Apply (Plan)'){
      		when{
              allOf {
              	environment name: 'ACTION', value: 'plan'
              }
            }
            steps {
                script {
                    def terraformPlanStatus = sh(
                        script: "terraform plan -detailed-exitcode -no-color -compact-warnings -var-file=variables.tfvars -parallelism=200 -out=plan",
                        returnStatus: true)
                        
                    sh 'terraform show -no-color plan > plan.pretty'
	
                    if (terraformPlanStatus == 2) {
						withCredentials([string(credentialsId: 'xxxxx', variable: 'SLACK_APP_TOKEN')]) {
							sh 'curl -F file=@plan.pretty -F "initial_comment=Changes on \\`xxxxxxx\\`" -F "filename=xxxxx`date +"%Y-%m-%d"`.txt" -F "filetype=text" -F channels=xxxxx -H "Authorization: Bearer ${SLACK_APP_TOKEN}" https://slack.com/api/files.upload'
                        }
                    }
                }
            }
        }
		
      	stage('Terraform Apply (Disable Alerts)'){
      		when{
              allOf {
              	environment name: 'ACTION', value: 'disable'
              }
            }
            steps {
                script {
					sh 'sed -i "s/enabled = true/enabled = false/g" variables.tfvars'
					
					def terraformApplyStatus = sh(
						script: "terraform apply -no-color -compact-warnings -var-file=variables.tfvars -parallelism=200 -auto-approve",
						returnStatus: true)
                        
                    if (terraformApplyStatus == 0) {
                    	slackSend channel: 'xxxxx_monitoring-valids', message: "`Disabled` Alerts on `${apps['xxxxx']}`", teamDomain: 'xxxxx', credentialsId: 'xxxxx'
                    } else {
                    	slackSend channel: 'xxxxx_monitoring-valids', message: "`Failed to disable` Alerts on `${apps['xxxxx']}`", teamDomain: 'xxxxx', credentialsId: 'xxxxx'
                    }
                }
            }
        }
      
      	stage('Terraform Apply (Enable Alerts)'){
      		when{
              allOf {
              	environment name: 'ACTION', value: 'enable'
              }
            }
            steps {
                script {
					def terraformApplyStatus = sh(
						script: "terraform apply -no-color -compact-warnings -var-file=variables.tfvars -parallelism=200 -auto-approve",
						returnStatus: true)
                        
                    if (terraformApplyStatus == 0) {
                    	slackSend channel: 'xxxxx-valids', message: "`Enabled` Alerts on `${apps['xxxxx']}`", teamDomain: 'xxxxx', credentialsId: 'xxxxx'
                    } else {
                    	slackSend channel: 'xxxxx-valids', message: "`Failed to Enable` Alerts on `${apps['xxxxx']}`", teamDomain: 'xxxxx', credentialsId: 'xxxxx'
                    }
                }
            }
        }
    }
    
    post {
        always {
            cleanWs()

            script {
                Jenkins.instance.queue.items.each {
                    if (it.isBlocked() && it.task.name == env.JOB_NAME){
                        Jenkins.instance.queue.cancel(it.task)
                    }
                }
            }
        }
    }
}
```

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
