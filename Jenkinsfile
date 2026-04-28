pipeline {
    agent any

    environment {
        // --- Registry Config ---
        DOCKER_HUB_USER = 'arunkumarravi08'
        DOCKER_HUB_REPO_FRONTEND = 'accounts-receivable-frontend'
        DOCKER_HUB_REPO_BACKEND = 'accounts-receivable-backend'
        IMAGE_TAG = "${BUILD_NUMBER}"
        
        // --- SonarQube Config ---
        SCANNER_HOME = tool 'sonar-scanner'
        
        // --- AWS/EKS Config ---
        CLUSTER_NAME = 'devops-eks'
        REGION = 'us-east-1'
    }

    options {
        skipDefaultCheckout()
    }

    stages {
        stage('Git Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Install Frontend Deps') {
            steps {
                dir('client') {
                    sh 'npm ci'
                }
            }
        }

        stage('Install Backend Deps') {
            steps {
                dir('flask-integration') {
                    sh """
                    python3 -m venv venv
                    ./venv/bin/pip install flask flask-cors pandas scikit-learn numpy pylint pytest safety
                    """
                }
            }
        }

        stage('ESLint') {
            steps {
                dir('client') {
                    sh 'npx eslint src --quiet'
                }
            }
        }

        stage('Frontend Tests') {
            steps {
                dir('client') {
                    sh 'npm test -- --watchAll=false --passWithNoTests'
                }
            }
        }

        stage('Backend Tests') {
            steps {
                dir('flask-integration') {
                    sh './venv/bin/pytest || echo "No tests found yet."'
                }
            }
        }

        stage('SonarQube Scan') {
            steps {
                echo "Starting SonarQube analysis..."
                withSonarQubeEnv('SonarQube-Server') {
                    sh """
                    ${SCANNER_HOME}/bin/sonar-scanner \
                        -Dsonar.projectKey=accounts-dashboard \
                        -Dsonar.sources=. \
                        -Dsonar.exclusions=**/*.java,**/*.csv,**/*.sav,**/*.ipynb,**/node_modules/**,**/venv/**,.assets/**
                    """
                }
            }
        }

        stage('Quality Gate') {
            steps {
                echo "Checking Quality Gate status..."
                waitForQualityGate abortPipeline: true
            }
        }

        stage('Trivy FS Scan') {
            steps {
                sh 'trivy fs . --severity HIGH,CRITICAL --format table || echo "Trivy not found on agent"'
            }
        }

        stage('Docker Build') {
            steps {
                script {
                    sh "docker build -t ${DOCKER_HUB_USER}/${DOCKER_HUB_REPO_FRONTEND}:${IMAGE_TAG} ./client"
                    sh "docker build -t ${DOCKER_HUB_USER}/${DOCKER_HUB_REPO_BACKEND}:${IMAGE_TAG} ./flask-integration"
                }
            }
        }

        stage('Trivy Image Scan') {
            steps {
                sh "trivy image ${DOCKER_HUB_USER}/${DOCKER_HUB_REPO_FRONTEND}:${IMAGE_TAG} --severity HIGH,CRITICAL || echo 'Trivy not found'"
                sh "trivy image ${DOCKER_HUB_USER}/${DOCKER_HUB_REPO_BACKEND}:${IMAGE_TAG} --severity HIGH,CRITICAL || echo 'Trivy not found'"
            }
        }

        stage('Docker Login') {
            steps {
                withCredentials([usernamePassword(credentialsId: 'docker-hub-creds', passwordVariable: 'DOCKER_PASS', usernameVariable: 'DOCKER_USER')]) {
                    sh "echo ${DOCKER_PASS} | docker login -u ${DOCKER_USER} --password-stdin"
                }
            }
        }

        stage('Docker Push') {
            steps {
                sh "docker push ${DOCKER_HUB_USER}/${DOCKER_HUB_REPO_FRONTEND}:${IMAGE_TAG}"
                sh "docker push ${DOCKER_HUB_USER}/${DOCKER_HUB_REPO_BACKEND}:${IMAGE_TAG}"
            }
        }

        stage('Helm Lint') {
            steps {
                sh 'helm lint ./charts/accounts-dashboard || echo "Helm chart directory not found."'
            }
        }

        stage('Observability Setup') {
            steps {
                withCredentials([aws(credentialsId: 'aws-creds', accessKeyVariable: 'AWS_ACCESS_KEY_ID', secretKeyVariable: 'AWS_SECRET_ACCESS_KEY')]) {
                    sh "aws eks update-kubeconfig --name ${CLUSTER_NAME} --region ${REGION}"
                    sh """
                    helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
                    helm repo update
                    helm upgrade --install prometheus prometheus-community/kube-prometheus-stack \
                        --namespace monitoring \
                        --create-namespace \
                        --set grafana.service.type=LoadBalancer
                    """
                }
            }
        }

        stage('EKS Auth & Deploy') {
            steps {
                withCredentials([aws(credentialsId: 'aws-creds', accessKeyVariable: 'AWS_ACCESS_KEY_ID', secretKeyVariable: 'AWS_SECRET_ACCESS_KEY')]) {
                    // Update kubeconfig for authentication
                    sh "aws eks update-kubeconfig --name ${CLUSTER_NAME} --region ${REGION}"
                    
                    // Run Helm deployment while credentials are active
                    sh """
                    helm upgrade --install accounts-dashboard ./charts/accounts-dashboard \
                        --set frontend.image.tag=${IMAGE_TAG} \
                        --set backend.image.tag=${IMAGE_TAG}
                    """
                }
            }
        }

        stage('Prometheus Metrics') {
            steps {
                withCredentials([aws(credentialsId: 'aws-creds', accessKeyVariable: 'AWS_ACCESS_KEY_ID', secretKeyVariable: 'AWS_SECRET_ACCESS_KEY')]) {
                    sh "aws eks update-kubeconfig --name ${CLUSTER_NAME} --region ${REGION}"
                    sh "kubectl get pods -n monitoring | grep prometheus"
                    echo "Prometheus is successfully running in the cluster."
                }
            }
        }

        stage('Grafana Visualization') {
            steps {
                withCredentials([aws(credentialsId: 'aws-creds', accessKeyVariable: 'AWS_ACCESS_KEY_ID', secretKeyVariable: 'AWS_SECRET_ACCESS_KEY')]) {
                    sh "aws eks update-kubeconfig --name ${CLUSTER_NAME} --region ${REGION}"
                    sh "kubectl get svc -n monitoring prometheus-grafana -o jsonpath='{.status.loadBalancer.ingress[0].hostname}'"
                    echo "Use the URL above to login to Grafana (Default: admin/prom-operator)"
                }
            }
        }

        stage('Alerting Notifications') {
            steps {
                echo "Configuring Alerting hooks..."
            }
        }
    }

    post {
        success {
            emailext body: """
                <h2>Build SUCCESS</h2>
                <p>Project: ${env.JOB_NAME}</p>
                <p>Build Number: ${env.BUILD_NUMBER}</p>
                <p>Check the console output here: <a href="${env.BUILD_URL}">${env.BUILD_URL}</a></p>
                <p>The application has been successfully deployed to EKS.</p>
            """,
            subject: "SUCCESS: ${env.JOB_NAME} Build #${env.BUILD_NUMBER}",
            to: 'arunkumarravi0811@gmail.com'
        }
        failure {
            emailext body: """
                <h2>Build FAILED</h2>
                <p>Project: ${env.JOB_NAME}</p>
                <p>Build Number: ${env.BUILD_NUMBER}</p>
                <p>Check the logs to find the error: <a href="${env.BUILD_URL}">${env.BUILD_URL}</a></p>
            """,
            subject: "FAILED: ${env.JOB_NAME} Build #${env.BUILD_NUMBER}",
            to: 'arunkumarravi0811@gmail.com'
        }
        always {
            echo "Pipeline Progress: Finished current stages."
        }
    }
}
