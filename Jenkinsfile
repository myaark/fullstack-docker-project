node {
    stage('Checkout') {
        echo '🔄 Checking out source code...'
        checkout scm
    }
    
    stage('Pre-build Cleanup & Check') {
        echo '🧹 Checking disk space and cleaning up...'
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
    
    stage('Verify Project Structure') {
        echo '📁 Verifying project structure...'
        sh '''
            echo "=== PROJECT STRUCTURE ==="
            ls -la
            
            echo "=== FRONTEND DIRECTORY ==="
            if [ -d frontend ]; then
                echo "✅ Frontend directory found"
                ls -la frontend/
                if [ -f frontend/Dockerfile ]; then
                    echo "✅ Frontend Dockerfile found"
                else
                    echo "❌ Frontend Dockerfile missing"
                    exit 1
                fi
            else
                echo "❌ Frontend directory missing"
                exit 1
            fi
            
            echo "=== BACKEND DIRECTORY ==="
            if [ -d backend ]; then
                echo "✅ Backend directory found"
                ls -la backend/
                if [ -f backend/Dockerfile ]; then
                    echo "✅ Backend Dockerfile found"
                else
                    echo "❌ Backend Dockerfile missing"
                    exit 1
                fi
            else
                echo "❌ Backend directory missing"
                exit 1
            fi
        '''
    }
    
    stage('Build Docker Images') {
        parallel(
            'Build Frontend': {
                echo '🐳 Building Frontend Docker image...'
                sh '''
                    cd frontend
                    echo "Building frontend image..."
                    docker build -t frontend-app:${BUILD_NUMBER} .
                    docker tag frontend-app:${BUILD_NUMBER} frontend-app:latest
                    echo "✅ Frontend image built successfully"
                '''
            },
            'Build Backend': {
                echo '🐳 Building Backend Docker image...'
                sh '''
                    cd backend
                    echo "Building backend image..."
                    docker build -t backend-app:${BUILD_NUMBER} .
                    docker tag backend-app:${BUILD_NUMBER} backend-app:latest
                    echo "✅ Backend image built successfully"
                '''
            }
        )
    }
    
    stage('Deploy Application') {
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
    
    stage('Application Health Check') {
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
    
    stage('Cleanup') {
        echo '🧹 Cleaning up...'
        sh '''
            # Clean up old images but keep latest
            docker image prune -f --filter "until=24h"
            echo "✅ Cleanup completed"
        '''
    }
}
