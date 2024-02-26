pipeline {
    agent any

    stages {
        stage('Prepare Environment') {
            steps {
                echo 'Initialize Environment'
                script {
                    def mavenHome = tool name: 'maven', type: 'maven'
                    env.MAVEN_HOME = mavenHome
                    env.PATH = "${mavenHome}/bin:${env.PATH}"
                    env.tag = "3.0"
                    env.dockerHubUser = "adeluyemi79"
                    env.containerName = "insure-me"
                    env.httpPort = "8081"
                }
            }
        }

        stage('Code Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Maven Build') {
            steps {
                sh "${env.MAVEN_HOME}/bin/mvn clean package"
            }
        }

        stage('Publish Test Reports') {
            steps {
                publishHTML([allowMissing: false, alwaysLinkToLastBuild: false, keepAll: false, reportDir: 'target/surefire-reports', reportFiles: 'index.html', reportName: 'HTML Report', reportTitles: '', useWrapperFileDirectly: true])
            }
        }

        stage('Docker Image Build') {
            steps {
                echo 'Creating Docker image'
                sh "docker build -t ${env.dockerHubUser}/${env.containerName}:${env.tag} --pull --no-cache ."
            }
        }

        stage('Docker Image Scan') {
            steps {
                echo 'Installing Trivy'
                sh "wget -qO /tmp/trivy.tar.gz https://github.com/aquasecurity/trivy/releases/download/v0.21.0/trivy_0.21.0_Linux-64bit.tar.gz"
                sh "tar -xzvf /tmp/trivy.tar.gz -C /tmp/"
                sh "sudo mv /tmp/trivy /usr/local/bin/"

                echo 'Scanning Docker image for vulnerabilities'
                sh "trivy ${env.dockerHubUser}/${env.containerName}:${env.tag}"
            }
        }

        stage('Publishing Image to DockerHub') {
            steps {
                echo 'Pushing the docker image to DockerHub'
                withCredentials([usernamePassword(credentialsId: 'dockerHubAccount', usernameVariable: 'dockerUser', passwordVariable: 'dockerPassword')]) {
                    sh "sudo docker login -u $dockerUser -p $dockerPassword"
                    sh "sudo docker push $dockerUser/$containerName:$tag"
                    echo "Image push complete"
                }
            }
        }

        stage('Docker Container Deployment') {
            steps {
                echo "Removing existing container: ${env.containerName}"
                sh "docker rm ${env.containerName} -f"
                
                echo "Pulling Docker image: ${env.dockerHubUser}/${env.containerName}:${env.tag}"
                sh "docker pull ${env.dockerHubUser}/${env.containerName}:${env.tag}"
                
                echo "Starting Docker container: ${env.containerName}"
                sh "docker run -d --rm -p ${env.httpPort}:${env.httpPort} --name ${env.containerName} ${env.dockerHubUser}/${env.containerName}:${env.tag}"
                
                echo "Application started on port: ${env.httpPort} (http)"
            }
        }
    }
}
