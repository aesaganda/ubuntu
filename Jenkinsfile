pipeline {
   agent any
   environment {
      REPOSITORY = 'docker.io/aesaganda' // Updated to use Docker Hub
      PCC_CONSOLE_URL = "twistlock1.garanti.lab:8083"
      CONTAINER_NAME = "ubuntu"
   }
   stages {
      stage('Clone repository') {
         steps {
            checkout scm
         }
      }      

      stage('Build') {
         steps {
            withCredentials([usernamePassword(credentialsId: 'dockerhub-creds', passwordVariable: 'DOCKER_PASS', usernameVariable: 'DOCKER_USER')]) {                
               sh ''' 
               docker login -u $DOCKER_USER -p $DOCKER_PASS $REPOSITORY
               echo "Building the Docker image..."
               docker build -t $REPOSITORY/$CONTAINER_NAME:$BUILD_NUMBER .
               docker image ls
               '''
            }
         }
      }

      stage('Container Scan') {
         steps {
            script {
               try {
                  prismaCloudScanImage ca: '', cert: '', dockerAddress: 'unix:///var/run/docker.sock', ignoreImageBuildTime: true, image: "$REPOSITORY/$CONTAINER_NAME:$BUILD_NUMBER", key: '', logLevel: 'debug', podmanPath: '', project: '', resultsFile: 'prisma-cloud-scan-results.json'
               } finally {
                  prismaCloudPublish resultsFilePattern: 'prisma-cloud-scan-results.json'
               }
            }
         }
      }         

      stage('Push Image') {
         steps {
            withCredentials([usernamePassword(credentialsId: 'docker_hub_creds', passwordVariable: 'DOCKER_PASS', usernameVariable: 'DOCKER_USER')]) {
               sh ''' 
               echo "Pushing the image to Docker Hub..."
               docker push $REPOSITORY/$CONTAINER_NAME:$BUILD_NUMBER
               '''
            }
         }
      }

      stage('Container Sandbox Scan') {
         steps {
            withCredentials([usernamePassword(credentialsId: 'ssh_creds', passwordVariable: 'SSH_PASS', usernameVariable: 'SSH_USER')]) {
               sh '''
               mkdir -p ~/.ssh/
               ssh-keyscan -t rsa,dsa twistlock1.garanti.lab >> ~/.ssh/known_hosts
               sshpass -p $SSH_PASS ssh $SSH_USER@twistlock1.garanti.lab 'bash -s' <<EOF         
               sudo chmod +x /home/sysadmin/apps/sandbox-scan.sh
               sudo PCC_CONSOLE_URL=$PCC_CONSOLE_URL token=$token CONTAINER_NAME=$CONTAINER_NAME TAG=$BUILD_NUMBER /home/sysadmin/apps/sandbox-scan.sh
               exit
               EOF
               '''
            }
         }
      }  
   }
}
