pipeline {
  agent {
    kubernetes {
      label 'docker-agent'
    }
  }
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
            container('docker') { // <-- Added container('docker') here
              withCredentials([usernamePassword(credentialsId: 'dockerhub-creds', passwordVariable: 'DOCKER_PASS', usernameVariable: 'DOCKER_USER')]) {          
                  sh '''  
                  docker login -u $DOCKER_USER -p $DOCKER_PASS $REPOSITORY
                  echo "Building the Docker image..."
                  docker build -t $REPOSITORY/$CONTAINER_NAME:$BUILD_NUMBER .
                  docker image ls
                  '''
              }
            } // <-- Closed container('docker') here
          }
      }

      stage('Container Scan') {
          steps {
            // Prisma Cloud Scan typically needs access to the Docker daemon.
            // If the plugin handles this automatically when run inside the 'docker' container,
            // this might just work. Otherwise, you might need specific configuration
            // within the 'prismaCloudScanImage' call for Docker daemon access.
            container('docker') { // <-- Added container('docker') here
              script {
                  try {
                    prismaCloudScanImage ca: '', cert: '', dockerAddress: 'unix:///var/run/docker.sock', ignoreImageBuildTime: true, image: "$REPOSITORY/$CONTAINER_NAME:$BUILD_NUMBER", key: '', logLevel: 'debug', podmanPath: '', project: '', resultsFile: 'prisma-cloud-scan-results.json'
                  } finally {
                    prismaCloudPublish resultsFilePattern: 'prisma-cloud-scan-results.json'
                  }
              }
            } // <-- Closed container('docker') here
          }
      }        

      stage('Push Image') {
          steps {
            container('docker') { // <-- Added container('docker') here
              withCredentials([usernamePassword(credentialsId: 'dockerhub-creds', passwordVariable: 'DOCKER_PASS', usernameVariable: 'DOCKER_USER')]) { // Corrected credential ID to match 'dockerhub-creds' if that's the one used in build stage
                  sh '''  
                  echo "Pushing the image to Docker Hub..."
                  docker push $REPOSITORY/$CONTAINER_NAME:$BUILD_NUMBER
                  '''
              }
            } // <-- Closed container('docker') here
          }
      }  
   }
}
