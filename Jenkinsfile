pipeline {
    agent any

    tools {
        jdk 'Java17'
        maven 'Maven3'
    }

    environment {
        DEPLOY_DIR = "/opt/eureka"
        EC2_HOST = "13.60.47.188"
        SERVICE_NAME = "eureka-server"
    }

    stages {
        stage('Checkout') {
            steps {
                git branch: 'master', url: 'https://github.com/Pransquare/Eureka.git'
            }
        }

        stage('Build') {
            steps {
                bat '''
                    echo ===== Building Eureka Service =====
                    mvn clean package -DskipTests
                '''
            }
        }

        stage('Deploy to EC2') {
            steps {
                echo "🚀 Deploying ${env.SERVICE_NAME} to EC2 (${env.EC2_HOST})"
                sshagent(['ec2-linux-key']) {
                    // We call Git Bash explicitly since Jenkins runs on Windows
                    bat '''
                        echo ===== Deploying to EC2 via SSH =====
                        "C:\\Program Files\\Git\\bin\\bash.exe" -c '
                            set -e
                            echo "=== Creating directories on EC2 ==="
                            ssh -o StrictHostKeyChecking=no ec2-user@13.60.47.188 "mkdir -p /opt/eureka/logs"
                            
                            echo "=== Copying JAR file to EC2 ==="
                            scp -o StrictHostKeyChecking=no target/eureka-server.jar ec2-user@13.60.47.188:/opt/eureka/
                            
                            echo "=== Stopping old Eureka instance (if any) ==="
                            ssh -o StrictHostKeyChecking=no ec2-user@13.60.47.188 "pkill -f eureka-server.jar || true"
                            
                            echo "=== Starting new Eureka service ==="
                            ssh -o StrictHostKeyChecking=no ec2-user@13.60.47.188 "nohup java -jar /opt/eureka/eureka-server.jar --server.port=8761 > /opt/eureka/logs/eureka.log 2>&1 &"
                            
                            echo "=== Deployment complete ==="
                        '
                    '''
                }
            }
        }
    }

    post {
        success {
            echo '✅ Eureka deployed successfully to EC2 (13.60.47.188)!'
        }
        failure {
            echo '❌ Deployment failed. Check Jenkins console logs for error details.'
        }
    }
}
