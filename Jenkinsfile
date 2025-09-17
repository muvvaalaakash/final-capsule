pipeline {
    agent any
    environment {
        DOCKER_IMAGE = "shaikhamadi/logistics-tracker-app"
        DOCKER_TAG   = "${BUILD_NUMBER}"
        EKS_CLUSTER_NAME = "hamadi-test-cluster"
        AWS_REGION = "ca-central-1"
    }
    stages {
        stage('Checkout') {
            steps {
                git credentialsId: 'github_credentials',
                    branch: 'main',
                    url: 'https://github.com/Hamadishaik/Final-capsule.git'
            }
        }
        stage('Build Docker Image') {
            steps {
                script {
                    echo "🛠️ Building Docker image..."
                    docker.build("${DOCKER_IMAGE}:${DOCKER_TAG}")
                    docker.build("${DOCKER_IMAGE}:latest")
                }
            }
        }
        stage('Docker Compose Test') {
            steps {
                script {
                    echo "🧪 Testing with Docker Compose..."
                    sh """
                        # Clean up existing containers
                        docker-compose down -v
                        # Build and start services
                        docker-compose build
                        docker-compose up -d
                        # Wait for services to be ready
                        sleep 30
                        docker-compose ps
                        # Test application health
                        i=1
                        while [ \$i -le 5 ]; do
                            if curl -f http://localhost:3000 2>/dev/null; then
                                echo "✅ Application is responding"
                                break
                            fi
                            echo "⏳ Waiting for application... attempt \$i/5"
                            i=\$((i+1))
                            sleep 10
                        done
                        # Show logs for debugging
                        docker-compose logs --tail=10 app
                    """
                }
            }
        }
        stage('Push Docker Image') {
            steps {
                script {
                    echo "📦 Pushing Docker image..."
                    docker.withRegistry('https://index.docker.io/v1/', 'dockerhub-credentials') {
                        sh """
                            docker push ${DOCKER_IMAGE}:${DOCKER_TAG}
                            docker push ${DOCKER_IMAGE}:latest
                        """
                    }
                    echo "✅ Docker images pushed successfully"
                }
            }
        }
        stage('Deploy to Kubernetes') {
            steps {
                withAWS(credentials: 'aws-eks-creds', region: "${AWS_REGION}") {
                    script {
                        sh """
                            echo "🔄 Updating kubeconfig..."
                            aws eks update-kubeconfig --region ${AWS_REGION} --name ${EKS_CLUSTER_NAME}
                            echo "🚀 Deploying to Kubernetes..."
                            # Deploy MongoDB first
                            kubectl apply -f mongodb-deployment.yaml
                            # Wait for MongoDB to be ready
                            echo "⏳ Waiting for MongoDB to be ready..."
                            # Check deployment status before waiting
                            kubectl get deployment mongodb
                            kubectl get pods -l app=mongodb
                            kubectl wait --for=condition=available --timeout=600s deployment/mongodb || {
                                echo "❌ MongoDB deployment failed - checking logs..."
                                kubectl describe deployment mongodb
                                kubectl get pods -l app=mongodb
                                kubectl logs -l app=mongodb --tail=50
                                exit 1
                            }
                            # Update application image and deploy
                            kubectl apply -f app-deployment.yaml
                            kubectl set image deployment/logistics-tracker-app \\
                                app=${DOCKER_IMAGE}:${DOCKER_TAG} --record
 

                            echo "⏳ Waiting for deployments to complete..."
                            kubectl rollout status deployment/mongodb --timeout=300s
                            kubectl rollout status deployment/logistics-tracker-app --timeout=300s
                            echo "📊 Deployment status:"
                            kubectl get deployments
                            kubectl get services
                            kubectl get pods
                        """
                    }
                }
            }
        }
        stage('Get LoadBalancer URL') {
            steps {
                withAWS(credentials: 'aws-eks-creds', region: "${AWS_REGION}") {
                    script {
                        sh """
                            echo "🌐 Getting LoadBalancer URL..."
                            aws eks update-kubeconfig --region ${AWS_REGION} --name ${EKS_CLUSTER_NAME}
                            i=1
                            while [ \$i -le 10 ]; do
                                EXTERNAL_IP=\$(kubectl get service logistics-tracker-service -o jsonpath='{.status.loadBalancer.ingress[0].ip}' 2>/dev/null || echo "")
                                EXTERNAL_HOSTNAME=\$(kubectl get service logistics-tracker-service -o jsonpath='{.status.loadBalancer.ingress[0].hostname}' 2>/dev/null || echo "")
                                if [ ! -z "\$EXTERNAL_IP" ]; then
                                    echo "🌐 Application URL: http://\$EXTERNAL_IP"
                                    break
                                elif [ ! -z "\$EXTERNAL_HOSTNAME" ]; then
                                    echo "🌐 Application URL: http://\$EXTERNAL_HOSTNAME"
                                    break
                                fi
                                echo "⏳ Waiting for LoadBalancer... attempt \$i/10"
                                i=\$((i+1))
                                sleep 20
                            done
                            # Show final service status
                            kubectl get service logistics-tracker-service
                            echo "✅ Deployment completed successfully!"
                        """
                    }
                }
            }
        }
    }
    post {
        success {
            echo "✅ Deployment successful!"
        }
        failure {
            echo "❌ Deployment failed!"
        }
    }
}