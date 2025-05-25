stage('Pre-build Cleanup & Check') {
    steps {
        script {
            sh '''
                echo "=== DISK SPACE CHECK ==="
                df -h /
                
                # Clean up any leftover Docker resources
                docker system prune -f
                
                # Check available space
                AVAILABLE=$(df / | tail -1 | awk '{print $4}')
                if [ "$AVAILABLE" -lt 1000000 ]; then
                    echo "❌ Insufficient disk space! Available: ${AVAILABLE}KB"
                    exit 1
                fi
                echo "✅ Sufficient space available: ${AVAILABLE}KB"
            '''
        }
    }
}

stage('Build Docker Images') {
    parallel {
        stage('Build Frontend') {
            steps {
                script {
                    echo '🐳 Building Frontend Docker image...'
                    sh '''
                        cd frontend
                        echo "Building frontend image..."
                        docker build -t frontend-app:${BUILD_NUMBER} .
                        docker tag frontend-app:${BUILD_NUMBER} frontend-app:latest
                        echo "✅ Frontend image built successfully"
                    '''
                }
            }
        }
        stage('Build Backend') {
            steps {
                script {
                    echo '🐳 Building Backend Docker image...'
                    sh '''
                        cd backend
                        echo "Building backend image..."
                        docker build -t backend-app:${BUILD_NUMBER} .
                        docker tag backend-app:${BUILD_NUMBER} backend-app:latest
                        echo "✅ Backend image built successfully"
                    '''
                }
            }
        }
    }
}

stage('Deploy Application') {
    steps {
        script {
            echo '🚀 Deploying application...'
            sh '''
                # Stop any existing containers
                docker stop frontend-container backend-container 2>/dev/null || true
                docker rm frontend-container backend-container 2>/dev/null || true
                
                # Start backend container
                docker run -d --name backend-container \
                    -p 8080:8080 \
                    --network bridge \
                    backend-app:latest
                
                # Start frontend container
                docker run -d --name frontend-container \
                    -p 3000:3000 \
                    --network bridge \
                    frontend-app:latest
                
                echo "✅ Application deployed successfully"
                echo "Frontend: http://localhost:3000"
                echo "Backend: http://localhost:8080"
            '''
        }
    }
}

stage('Application Health Check') {
    steps {
        script {
            echo '🏥 Performing health checks...'
            sh '''
                # Wait a moment for containers to start
                sleep 10
                
                # Check if containers are running
                echo "=== CONTAINER STATUS ==="
                docker ps --filter name=frontend-container --filter name=backend-container
                
                # Check frontend health
                echo "=== FRONTEND HEALTH CHECK ==="
                if curl -f http://localhost:3000 >/dev/null 2>&1; then
                    echo "✅ Frontend is healthy"
                else
                    echo "⚠️ Frontend health check failed"
                    docker logs frontend-container --tail 20
                fi
                
                # Check backend health
                echo "=== BACKEND HEALTH CHECK ==="
                if curl -f http://localhost:8080 >/dev/null 2>&1; then
                    echo "✅ Backend is healthy"
                else
                    echo "⚠️ Backend health check failed"
                    docker logs backend-container --tail 20
                fi
            '''
        }
    }
}
