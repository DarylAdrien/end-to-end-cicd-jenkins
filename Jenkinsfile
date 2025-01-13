pipeline {
    agent {
		docker {
			image 'jenkins-image:latest'  // Docker image that contains Docker tools (custom image)
			args '--user root -v /var/run/docker.sock:/var/run/docker.sock'  // Mount Docker socket to enable Docker-in-Docker
		}
    }
    options{
		skipDefaultCheckout(true)
    }
    triggers {
		githubPush()  
    }
	
    environment {
        DOCKER_IMAGE = "daryladriene/flask-app-linux"  
        DOCKER_TAG = "v${BUILD_NUMBER}"
        K8S_DEPLOYMENT_FILE = 'demo/deployment.yaml' 
        GIT_REPO_MAIN_URL = 'https://github.com/DarylAdrien/end-to-end-cicd-jenkins.git' 
        GIT_REPO_URL = 'https://github.com/DarylAdrien/cicd-k8s-manifests.git' 
        GIT_BRANCH = 'main' 
		GIT_USER_NAME = "DarylAdrien" 
		GIT_REPO_NAME = "cicd-k8s-manifests"
    }

    stages {
	    
		stage("Clean Up") {

			steps {

				sh 'rm -rf *'
			
			}

		}
			
		stage('Checkout Main Repo') {
			steps {
				
				dir('repo1') { 
					checkout([
						$class: 'GitSCM',
						branches: [[name: '*/main']],
						userRemoteConfigs: [[url: env.GIT_REPO_MAIN_URL, credentialsId: 'f1447528-919f-41bb-8ee3-0a1778b85f46']]
					])
				}

			}
		}
			
		stage('Static Code Analysis') {

			steps {

				script {
					withCredentials([usernamePassword(credentialsId: 'sonarqube',usernameVariable: 'SONARQUBE_USER',passwordVariable: 'SONARQUBE_TOKEN')]) {
						sh '''
							cd repo1
							sonar-scanner \
								-Dsonar.projectKey=my-flask-app \
								-Dsonar.sources=./application \
								-Dsonar.host.url=http://172.17.0.1:9000 \
								-Dsonar.login=$SONARQUBE_TOKEN
						'''

					}
				}
			}
		}


        stage('Build Docker Image') {

            steps {

                script {
                    sh '''
						echo "Building Docker Image..."
		    			cd repo1
                        docker build -t $DOCKER_IMAGE:$DOCKER_TAG .
                    '''
                }

            }
        }

        stage('Push Docker Image') {

            steps {

                withCredentials([usernamePassword(credentialsId: 'a50c698c-ff32-4319-a393-abc15e71fd75', usernameVariable: 'DOCKER_USERNAME', passwordVariable: 'DOCKER_PASSWORD')]) {
                    script {

                        sh '''
							echo "Logging into Docker Hub..."
                            docker login -u $DOCKER_USERNAME -p $DOCKER_PASSWORD
                        '''
                        sh '''
                            echo "Pushing Docker image to Docker Hub..."
                            docker push $DOCKER_IMAGE:$DOCKER_TAG
                        '''
                    }
                }

            }
        }

	    
		stage('Checkout Manifest Repo') {

            steps {

				dir('repo2') {
	                checkout([
	                    $class: 'GitSCM',
	                    branches: [[name: '*/main']],
	                    userRemoteConfigs: [[url: env.GIT_REPO_URL, credentialsId: 'f1447528-919f-41bb-8ee3-0a1778b85f46']]
	                ])
				}

	    	}
        }

	    
		stage('Update Kubernetes Manifest') {

			steps {

				script {
					sh """
						echo "Updating Kubernetes deployment with new Docker image tag..."
						cd repo2
						sed -i 's|$DOCKER_IMAGE:.*|$DOCKER_IMAGE:$DOCKER_TAG|' $K8S_DEPLOYMENT_FILE
						echo "Updated deployment.yaml"
					"""
				}

			}
		}
	
		stage('Commit and Push Changes to Git') {

			steps {

				withCredentials([usernamePassword(credentialsId: 'f1447528-919f-41bb-8ee3-0a1778b85f46', usernameVariable: 'GIT_USERNAME', passwordVariable: 'GIT_TOKEN')]) {
					script {

						sh '''
							echo "Committing and pushing changes to Git repository..."
							cd repo2
							git config --global --add safe.directory /var/lib/jenkins/workspace/First-pipeline/repo2
							git config  user.email "edaryl2705@gmail.com"  
							git config  user.name "DarylAdrien" 
							git add .
							git commit -m "Update Docker image tag to $DOCKER_IMAGE:$DOCKER_TAG" || echo "No changes to commit."
							git push https://$GIT_TOKEN@github.com/$GIT_USER_NAME/$GIT_REPO_NAME.git HEAD:$GIT_BRANCH
						'''

					}
				}

			}
		}
	    
    }

    post {

        success {
            echo "Pipeline completed successfully!"
        }

        failure {
            echo "Pipeline failed!"
        }

    }
}
