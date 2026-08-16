pipeline {
    agent any

    environment {
        JAVA_HOME  = "/opt/homebrew/Cellar/openjdk/26.0.2/libexec/openjdk.jdk/Contents/Home"
        NODE_HOME  = "/opt/homebrew/opt/node@20/bin"
        PATH       = "${NODE_HOME}:${JAVA_HOME}/bin:${env.PATH}"
        IMAGE_TAG        = "${env.BUILD_NUMBER}"
        BACKEND_IMAGE    = "corona-tracker-backend"
        FRONTEND_IMAGE   = "corona-tracker-frontend"
        K8S_NAMESPACE    = "corona-tracker-app-namespace"
    }

    stages {

        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Backend Build') {
            steps {
                dir('corona-tracker-backend') {
                    sh 'mvn clean compile'
                }
            }
        }

        stage('Backend Test') {
            steps {
                dir('corona-tracker-backend') {
                    sh 'mvn test'
                }
            }
            post {
                always {
                    junit 'corona-tracker-backend/target/surefire-reports/*.xml'
                }
            }
        }

        stage('Backend Package') {
            steps {
                dir('corona-tracker-backend') {
                    sh 'mvn package -DskipTests'
                }
            }
        }

        stage('Frontend Install & Test') {
            steps {
                dir('corona-tracker-frontend') {
                    sh 'npm ci --legacy-peer-deps'
                    sh 'CI=true npm test -- --watchAll=false'
                }
            }
        }

        stage('Frontend Build') {
            steps {
                dir('corona-tracker-frontend') {
                    sh 'npm run build'
                }
            }
        }

        stage('Docker Build (into Minikube daemon)') {
            steps {
                sh '''
                    eval $(minikube docker-env)
                    docker build -t ${BACKEND_IMAGE}:${IMAGE_TAG} -t ${BACKEND_IMAGE}:latest ./corona-tracker-backend
                    docker build -t ${FRONTEND_IMAGE}:${IMAGE_TAG} -t ${FRONTEND_IMAGE}:latest ./corona-tracker-frontend
                '''
            }
        }

        stage('Deploy to Minikube') {
            steps {
                sh '''
                    kubectl create namespace ${K8S_NAMESPACE} --dry-run=client -o yaml | kubectl apply -f -
                    kubectl apply -f kubernetes/ -n ${K8S_NAMESPACE}
                    kubectl set image deployment/corona-tracker-backend-deploy corona-tracker-backend=${BACKEND_IMAGE}:${IMAGE_TAG} -n ${K8S_NAMESPACE} --record || true
                    kubectl set image deployment/corona-tracker-frontend-deploy corona-tracker-frontend=${FRONTEND_IMAGE}:${IMAGE_TAG} -n ${K8S_NAMESPACE} --record || true
                    kubectl rollout status deployment/corona-tracker-backend-deploy -n ${K8S_NAMESPACE} --timeout=120s
                    kubectl rollout status deployment/corona-tracker-frontend-deploy -n ${K8S_NAMESPACE} --timeout=120s
                '''
            }
        }
    }

    post {
        success {
            echo "Pipeline succeeded — build #${env.BUILD_NUMBER} deployed to Minikube."
        }
        failure {
            echo "Pipeline failed — check stage logs above."
        }
        always {
            cleanWs()
        }
    }
}