pipeline {
    agent any

    environment {
        DOCKER_CREDENTIALS_ID = 'roseaw-dockerhub'
        DOCKER_IMAGE = 'cithit/minterjt'                                                                    //<------change this
        IMAGE_TAG = "lab37-build-${BUILD_NUMBER}"
        GITHUB_URL = 'https://github.com/srxxm49mps-png/225-lab3-7.git'                                          //<------change this
        KUBECONFIG = credentials('minterjt')                                                         //<------change this
    }

    stages {
        stage('Checkout') {
            steps {
                sh 'test "$(kubectl get nodes -o jsonpath=\'{.items[0].metadata.name}\')" = "minterjt-fl26-pool1-sfjhj-t2bf2"'
                cleanWs()
                checkout([$class: 'GitSCM', branches: [[name: '*/main']],
                          userRemoteConfigs: [[url: "${GITHUB_URL}"]]])
            }
        }

        stage('Configure Assigned LoadBalancer IP') {
            steps {
                sh 'kubectl apply -f metallb-production.yaml'
            }
        }

        stage('Lint HTML') {
            steps {
                sh 'npm install htmlhint --save-dev'
                sh 'npx htmlhint *.html'
            }
        }

        stage('Build Docker Image') {
            steps {
                script {
                    docker.withRegistry('https://registry-1.docker.io', 'roseaw-dockerhub') {
                        docker.build("${DOCKER_IMAGE}:${IMAGE_TAG}", "-f Dockerfile.build .")
                    }
                }
            }
        }
        stage('Push Docker Image') {
            steps {
                script {
                    docker.withRegistry('https://index.docker.io/v1/', "${DOCKER_CREDENTIALS_ID}") {
                        docker.image("${DOCKER_IMAGE}:${IMAGE_TAG}").push()
                    }
                }
            }
        }

        stage('Deploy to Dev Environment') {
            steps {
                script {
                    // Set up Kubernetes configuration using the specified KUBECONFIG
                    // Update deployment-dev.yaml to use the new image tag
                    sh "sed -i 's|${DOCKER_IMAGE}:latest|${DOCKER_IMAGE}:${IMAGE_TAG}|' deployment-dev.yaml"
                    sh "kubectl apply -f deployment-dev.yaml"
                    sh "kubectl rollout status deployment/minterjt-lab37-dev --timeout=180s"
                }
            }
        }

        stage("Run Acceptance Tests") {
            steps {
                script {
                    sh 'docker build -t minterjt-lab37-tests -f Dockerfile.test .'
                    sh 'docker run --rm minterjt-lab37-tests'
                }
            }
        }

        stage('Run Security Checks') {
            steps {
                sh 'curl --fail http://10.48.229.166/lab3-7/'
                sh 'docker pull public.ecr.aws/portswigger/dastardly:latest'
                sh '''
                    docker run --rm --user $(id -u) -v ${WORKSPACE}:${WORKSPACE}:rw \
                    -e BURP_START_URL=http://10.48.229.166/lab3-7/ \
                    -e BURP_REPORT_FILE_PATH=${WORKSPACE}/dastardly-report.xml \
                    public.ecr.aws/portswigger/dastardly:latest
                '''
            }
        }

        stage('Deploy to Prod Environment') {
            steps {
                script {
                    // Set up Kubernetes configuration using the specified KUBECONFIG
                    //sh "ls -la"
                    sh "sed -i 's|${DOCKER_IMAGE}:latest|${DOCKER_IMAGE}:${IMAGE_TAG}|' deployment-prod.yaml"
                    sh "kubectl apply -f deployment-prod.yaml"
                    sh "kubectl rollout status deployment/minterjt-lab37-prod --timeout=180s"
                    sh "kubectl wait --for=jsonpath='{.status.loadBalancer.ingress[0].ip}'=10.48.229.34 service/minterjt-lab37-prod-service --timeout=120s"
                    sh 'curl --fail --retry 10 --retry-delay 3 --retry-connrefused http://10.48.229.34:8083/ | grep "Lab 3-7 Works"'
                }
            }
        }
        stage('Check Kubernetes Cluster') {
            steps {
                script {
                    sh "kubectl get all"
                }
            }
        }
    }
    post {
        always {
            junit testResults: 'dastardly-report.xml', skipPublishingChecks: true, allowEmptyResults: true
            archiveArtifacts artifacts: 'dastardly-report.xml', allowEmptyArchive: true
        }

        success {
            slackSend color: "good", message: "Build Completed: ${env.JOB_NAME} ${env.BUILD_NUMBER}"
        }
        unstable {
            slackSend color: "warning", message: "Build Unstable: ${env.JOB_NAME} ${env.BUILD_NUMBER}"
        }
        failure {
            slackSend color: "danger", message: "Build Failed: ${env.JOB_NAME} ${env.BUILD_NUMBER}"
        }
    }
}

