pipeline { 
	agent {
		label {
		label "Jenkins-Slave"
		}
	}
    environment { 
        registry = "itquery/aetna-bsafe-docker" 
        registryCredential = 'dockerhub_cred' 
        dockerImage = 'docker.build registry + ":$BUILD_NUMBER"' 
	}

    stages { 
        stage('Code Checkout') { 
	    steps { 
		git branch: 'main',
		url: 'https://github.com/itquery/Aetna-BSafe.git'
		}
        } 

	stage('Clean & Package') {
	steps {
	    dir('complete') {
	    sh "./mvnw clean package"
		    }
		}
	}

	stage('Test Result') {
	steps {
	    dir('complete') {
	    junit '**/target/surefire-reports/TEST-*.xml'
	    archiveArtifacts artifacts: 'target*//*.jar', followSymlinks: false, onlyIfSuccessful: true
		    }
		}
	}

        stage('Build Docker Image') { 
	    steps { 
	        dir('complete') {
                script { 
                    dockerImage = docker.build registry + ":$BUILD_NUMBER" 
                }
	        }
            } 
        }
        stage('Push Docker Image') { 
            steps { 
                script { 
		 docker.withRegistry('https://registry.hub.docker.com', 'dockerhub_cred') {
		 dockerImage.push()   
			}
		} 
           }
        } 

	stage("Run Docker Container") {
		steps {
		// Stop running container to avoid conflict
                script{
                    def doc_containers = sh(returnStdout: true, script: 'docker container ps -aq ').replaceAll("\n", " ") 
                    if (doc_containers) {
                        sh "docker stop ${doc_containers}"
                    }
                }

			sh "docker run -i --name BSafe_$BUILD_NUMBER -d -p 3000:8080 $registry:$BUILD_NUMBER --bind 0.0.0.0"
			sh "docker ps -f name=BSafe_$BUILD_NUMBER"
	          }
		}

	stage("Capture HTTP response") {
            steps {
                  script {
		  // Give some time to tomcat application inside the container to start
			sh "sleep 5"
			sh "curl -X GET -s GET http://localhost:3000/"
                }
            }
        }
        
	stage('Cleaning up') { 
		steps { 
		sh "docker stop BSafe_$BUILD_NUMBER"  
		sh "docker rm   BSafe_$BUILD_NUMBER"    
		sh "docker rmi $registry:$BUILD_NUMBER" 

			}
		}
    }
        
	post {
		always {
			script {
			    def mailRecipients = 'rajesh.maurya@gmail.com'
			    def jobName = currentBuild.fullDisplayName
			    emailext body: '''${SCRIPT, template="groovy-html.template"}''',
			    mimeType: 'text/html',
			    subject: "[Jenkins] ${jobName}",
			    to: "${mailRecipients}",
			    replyTo: "${mailRecipients}",
			    recipientProviders: [[$class: 'CulpritsRecipientProvider']]
			}
		}
	}

}
