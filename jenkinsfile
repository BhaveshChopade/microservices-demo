pipeline {
    agent any

    environment {
        // --- CONFIGURATION ---
        DOCKERHUB_USER = bhaveshhhhh  // <--- CHANGE THIS
        APP_NAME = 'frontend'
        IMAGE_TAG = "${BUILD_NUMBER}"
        
        // SonarCloud Keys (From your message)
        SONAR_ORG = 'bhaveshchopade'
        SONAR_PROJECT = 'BhaveshChopade_microservices-demo'
        
        // GitOps Config
        MANIFEST_FILE = 'release/kubernetes-manifests.yaml'
        REPO_URL = 'github.com/BhaveshChopade/microservices-demo.git' // Check if this matches your user
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Static Code Analysis (SonarCloud)') {
            steps {
                script {
                    // Use the tool we configured in Jenkins Global Tools
                    def scannerHome = tool 'sonar-scanner'
                    
                    // Run the scan
                    withSonarQubeEnv('sonarcloud') {
                        // We only scan src/frontend to save time and focus on the relevant code
                        sh "${scannerHome}/bin/sonar-scanner \
                        -Dsonar.organization=${SONAR_ORG} \
                        -Dsonar.projectKey=${SONAR_PROJECT} \
                        -Dsonar.sources=src/frontend \
                        -Dsonar.host.url=https://sonarcloud.io"
                    }
                }
            }
        }

        stage('Security Scan (Trivy FS)') {
            steps {
                script {
                    // Scan the Go source code for vulnerabilities
                    sh "trivy fs --exit-code 0 --severity HIGH,CRITICAL src/frontend"
                }
            }
        }

        stage('Build Docker Image') {
            steps {
                script {
                    dir('src/frontend') {
                        // Build with the Build Number (e.g., frontend:5)
                        sh "docker build -t ${DOCKERHUB_USER}/${APP_NAME}:${IMAGE_TAG} ."
                        // Also tag as latest
                        sh "docker tag ${DOCKERHUB_USER}/${APP_NAME}:${IMAGE_TAG} ${DOCKERHUB_USER}/${APP_NAME}:latest"
                    }
                }
            }
        }

        stage('Image Scan (Trivy)') {
            steps {
                script {
                    // Scan the final Docker image before pushing
                    // We exit 0 (don't fail) so you can see the results, but you can change to 1 to enforce security
                    sh "trivy image --exit-code 0 --severity HIGH,CRITICAL ${DOCKERHUB_USER}/${APP_NAME}:${IMAGE_TAG}"
                }
            }
        }

        stage('Push to Registry') {
            steps {
                script {
                    // Login using the credentials we stored in Jenkins
                    withCredentials([usernamePassword(credentialsId: 'dockerhub-login', passwordVariable: 'DOCKER_PASS', usernameVariable: 'DOCKER_USER')]) {
                        sh "echo $DOCKER_PASS | docker login -u $DOCKER_USER --password-stdin"
                        sh "docker push ${DOCKERHUB_USER}/${APP_NAME}:${IMAGE_TAG}"
                        sh "docker push ${DOCKERHUB_USER}/${APP_NAME}:latest"
                    }
                }
            }
        }

        stage('Update Manifest (GitOps)') {
            steps {
                script {
                    withCredentials([usernamePassword(credentialsId: 'github-pat', passwordVariable: 'GIT_TOKEN', usernameVariable: 'GIT_USER')]) {
                        // Configure Git
                        sh 'git config user.email "jenkins@pipeline.com"'
                        sh 'git config user.name "Jenkins Pipeline"'
                        
                        // MAGIC: Use sed to find the old image tag and replace it with the new one
                        // We look for "image: */frontend*" and replace the whole line
                        sh "sed -i 's|image: .*/frontend.*|image: ${DOCKERHUB_USER}/${APP_NAME}:${IMAGE_TAG}|g' ${MANIFEST_FILE}"
                        
                        // Commit and Push
                        sh "git add ${MANIFEST_FILE}"
                        sh "git commit -m 'Update frontend image to tag ${IMAGE_TAG}'"
                        sh "git push https://${GIT_TOKEN}@${REPO_URL} HEAD:main"
                    }
                }
            }
        }
    }
    
    post {
        always {
            // Cleanup to save disk space
            sh 'docker logout || true'
            cleanWs()
        }
    }
}
