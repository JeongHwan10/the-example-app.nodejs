pipeline {
	agent any
	
	triggers {
		pollSCM 'H/1 * * * *'
	}
	
	options {
		timestamps()
	}
	
	environment {
		// 이미지 이름
		ProjectName='git-nodejs'
		
		DockerUserName='wjdghks1057'
		registryUrl = "https://docker.io/"
		registryCredential = 'docker-hub'
	}
	
	stages {
		stage('Checkout') {
			 environment {
				 /* Checkout environment */
				 gitCredentialsId = 'gitToken'
				 githubUrl = 'https://github.com/JeongHwan10/the-example-app.nodejs.git'
				 gitBranchName = 'master'
				 name = ''
				 refspec = ''
			 }
		 
		 steps {
			 checkout changelog: true, poll: true, scm: [
				 $class: 'GitSCM',
				 branches: [
					 [
						 name: "${env.gitBranchName}"
					 ]
				 ],
				 extensions: [], 
				 userRemoteConfigs: [
					 [
						 credentialsId: "${env.gitCredentialsId}",
						 name: "",
						 refspec: "",
						 url: "${env.githubUrl}"
					 ]
				 ]
			 ]
		 }
		 
		 post {
			 failure {
				 script { env.FAILURE_STAGE = 'Checkout' }
			 }
		 }
	 }
	    
	    stage("NodeJS Build") {
            steps {
                nodejs(nodeJSInstallationName: 'nodejs 17.5.0', configId: '') {
                    sh '''
						npm install
					'''
                }
    	    }
	    }
		
		stage('Docker Build image and push') {
			environment {
				dockerFile = 'Dockerfile'
				imageTag = "${BUILD_NUMBER}"
				buildContext = ''
				
				DOCKER_BUILDKIT = 0
				COMPOSE_DOCKER_CLI_BUILD = 0
			}
			
			steps {
				sh 'docker build -t ${DockerUserName}/${ProjectName}:${imageTag} -f ${dockerFile} ${WORKSPACE}/${buildContext}'
				
				withDockerRegistry([credentialsId: registryCredential, url: "${registryUrl}"]) {
					sh 'docker tag $DockerUserName/$ProjectName:$imageTag $DockerUserName/$ProjectName:latest'
					sh 'docker push $DockerUserName/$ProjectName:$imageTag'
					sh 'docker push $DockerUserName/$ProjectName:latest'
				}
			}
			post {
				always { 
					sh "docker logout"
					
					sh "echo Docker image Clean..."
					sh 'docker rmi $DockerUserName/$ProjectName:$imageTag'
					// sh 'docker rmi $DockerUserName/$ProjectName:latest'
				}
				failure {
					script { env.FAILURE_STAGE = 'Docker' }
				}
			}
		}
		
		stage('Deploy to k8s') {
			environment {
				KubeconfigId = 'Kubeconfig'
				yamlFilePath = 'Deployment.yaml'
			}
			
			steps {
				kubernetesDeploy(
					kubeconfigId: "${KubeconfigId}",
					configs: "${yamlFilePath}",
					enableConfigSubstitution: true,
					
					dockerCredentials: [[
						credentialsId: "${registryCredential}", url: "${registryUrl}"
					]]
                )
			}
			post {
				failure {
					script { env.FAILURE_STAGE = 'Deploy to k8s' }
				}
			}
		}
	}
	
	post {
		always {
			script {
				slackSend(
					tokenCredentialId: 'slackJenkinsId',
					color: (currentBuild.currentResult == 'SUCCESS') ? 'good' : 'danger',
					message: "${currentBuild.fullDisplayName} - `${currentBuild.currentResult}`\n${env.BUILD_URL}"
				)
			}
		}
	}
}