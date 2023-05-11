pipeline {
    agent any
  
    environment {
       IMAGE_NAME = "roberoi38/service_jenkins"
       IMAGE_VERSION ="3.0.0"
       registryCredential = 'dockerhublogin'
       COSIGN_PRIVATE_KEY= credentials('cosign-private-key')
       COSIGN_PASSWORD= credentials('cosign_password')
       CLOUDSDK_CORE_PROJECT = "focused-airline-379307"
       GCLOUD_CREDS = credentials('gcloud-creds')
       GKE_CLUSTER = 'my-kubectl-cluster'
       GKE_ZONE = 'us-central1-c'
       
    }
  
    stages {
        stage('SNYK test') {
           steps {
                dir('/polo/spring-security-react-ant-design-polls-app/polling-app-server') {
                    snykSecurity(
                        snykInstallation: 'snyk-scanner',
                        snykTokenId: 'snyk-api-token',
                        failOnIssues: 'false',
                        failOnError: 'false',
                    )
                }
            }
        }
        stage('SonarQube SAST') {
            steps {
                dir('/polo/spring-security-react-ant-design-polls-app/polling-app-server') {
                    withSonarQubeEnv('sonarqube-scanner'){
                        sh 'mvn sonar:sonar -Dsonar.exclusions=**/*.java'
                        
                    }
                }
            }
        }
        stage('Create container image') {
          steps {
              script {
                  dir('/polo/spring-security-react-ant-design-polls-app/polling-app-client') {
                      dockerImage = docker.build(IMAGE_NAME)
                  }
              }
          }
      }
      stage('Scan image grype') {
          steps {
              script {
                  dir('/polo/spring-security-react-ant-design-polls-app/polling-app-client') {
                      sh "grype ${IMAGE_NAME} --scope AllLayers"
                      //sh "grype ${dockerimagename} --scope AllLayers --fail-on=critical"
                  }
              }
          }
      }
      stage('Tag container image') {
          steps {
              sh "docker tag docker.io/${IMAGE_NAME} $IMAGE_NAME:$IMAGE_VERSION"
          }
      }
      stage('Pushing Image') {
          environment {
              registryCredential = 'dockerhublogin'
          }
          steps {
              script {
                  docker.withRegistry('https://registry.hub.docker.com', registryCredential) {
                      dockerImage.push('3.0.0')
                  }
              }
          }
      }
      stage('Container Signin Cosign') {
      steps {
      withCredentials([usernamePassword(credentialsId: 'dockerhublogin', usernameVariable: 'DOCKERHUB_USERNAME', passwordVariable: 'DOCKERHUB_PASSWORD')]) {
        sh "docker login -u ${DOCKERHUB_USERNAME} -p ${DOCKERHUB_PASSWORD}"
        sh "cosign sign --key $COSIGN_PRIVATE_KEY docker.io/$IMAGE_NAME:$IMAGE_VERSION -y"
               }
           }
      }
      stage('Kube-score') {
        steps {
          dir('/polo/spring-security-react-ant-design-polls-app/deployments') {
             sh '''
              kube-score score polling-app-client.yaml || true
             '''
              }
          }
       }
       stage('check gcloud version') {
          steps {
          dir('/polo/spring-security-react-ant-design-polls-app/deployments') {
          sh '''
          gcloud auth activate-service-account --key-file=${GCLOUD_CREDS}
          gcloud container clusters get-credentials ${GKE_CLUSTER} --zone ${GKE_ZONE}
          kubectl get pods
          kubectl apply -f polling-app-client.yaml
          '''
          }
          }
        }
    }
}
