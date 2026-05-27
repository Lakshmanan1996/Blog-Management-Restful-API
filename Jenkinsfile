pipeline 
{

    agent none

    tools {
        maven 'Maven'
        
        
    }

    environment {

        IMAGE1       = "blogapi"
        
        DOCKERHUB_USER = "lakshvar96"
        GIT_REPO = "https://github.com/5OBHAN/Blog-Management-Restful-API.git"
    }
    
   /* =====================================================   
   CHECKOUT
    ===================================================== */

    stages {

        stage('Checkout Code') {
            agent { label 'workernode1' }
            steps {
                checkout([$class: 'GitSCM',
                    branches: [[name: 'main']],
                    userRemoteConfigs: [[url: "${GIT_REPO}"]]
                ])
            }
        }


        stage('Stash Source') {
            agent { label 'workernode1' }
            steps {
                stash includes: '**/*', name: 'source-code'
            }
        }


        /* ===================== Build Maven Stage ===================== */
        stage('Build') {
            agent { label 'workernode1' }
            
                    
            steps {
                unstash 'source-code'
                sh 'mvn clean package -DskipTests'
            }
        }

        /* =====================================================
           SONARQUBE ANALYSIS
        ===================================================== */

        stage('SonarQube Analysis') {
            agent { label 'workernode2' }
            steps {
                unstash 'source-code'
                script {
                    def scannerHome = tool 'SonarQubeScanner'
                    
                    withSonarQubeEnv('sonarqube') {
                        
                        sh """
                             mvn clean verify sonar:sonar \
                             -DskipTests \
                             -Dsonar.projectKey=Blog-API \
                             -Dsonar.projectName=Blog-API \
                        """
                    }
                        
                }
            }
        }
        

        /* =====================================================
           QUALITY GATE
        ===================================================== */

        stage('Quality Gate') {
            agent { label 'workernode2' }
            steps {
                timeout(time: 2, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: true
                }
            }
        }

        

        /* =====================================================
           DOCKER BUILD
        ===================================================== */

        stage('Docker Build') {
            agent { label 'workernode3' }
            steps {
                unstash 'source-code'
                
                echo "Build a image for Blog-API"
                
                
                sh """
                docker build -t ${DOCKERHUB_USER}/${IMAGE1}:${BUILD_NUMBER} .
                docker tag ${DOCKERHUB_USER}/${IMAGE1}:${BUILD_NUMBER} ${DOCKERHUB_USER}/${IMAGE1}:latest 
                """
            }
                
                
        }
        

        /* =====================================================
           TRIVY IMAGE SCAN
        ===================================================== */

        stage('Trivy Scan') {
            agent { label 'workernode3' }
            steps {
                sh """
                trivy image --exit-code 0 --severity HIGH,CRITICAL ${DOCKERHUB_USER}/${IMAGE1}:${BUILD_NUMBER}
                
                """
            }
        }

         /* =====================================================
           DOCKER PUSH
        ===================================================== */

        stage('Push Image') {
            agent { label 'workernode3' }
            steps {
                withCredentials([usernamePassword(
                    credentialsId: 'dockerhub-creds',
                    usernameVariable: 'DOCKER_USER',
                    passwordVariable: 'DOCKER_PASS'
                )]) {
                    sh 'echo $DOCKER_PASS | docker login -u $DOCKER_USER --password-stdin'
                }

                sh """
                docker push ${DOCKERHUB_USER}/${IMAGE1}:${BUILD_NUMBER}
                docker push ${DOCKERHUB_USER}/${IMAGE1}:latest
                """

                 
                     
            }
        }

    }    
    post {
        success {
            echo "✅ Blog-API CI Pipeline SUCCESS"
        }
        failure {
            echo "❌ Blog-API CI Pipeline FAILED"
        }
    }
}

