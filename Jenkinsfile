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
        PEM_PATH = "C:\\Users\\KRISHNA\\Downloads\\krishna.pem" // <-- your PEM file path
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
                
                // Direct SSH & SCP using PEM key
                bat """
                    echo ===== Deploying to EC2 via SSH =====
                    "C:\\Program Files\\Git\\bin\\bash.exe" -c '
                        set -e
                        echo "=== Creating directories on EC2 ==="
                        ssh -i ${env.PEM_PATH} -o StrictHostKeyChecking=no ec2-user@${env.EC2_HOST} "mkdir -p ${env.DEPLOY_DIR}/logs"
                        
                        echo "=== Copying JAR file to EC2 ==="
                        scp -i ${env.PEM_PATH} -o StrictHostKeyChecking=no target/${env.SERVICE_NAME}.jar ec2-user@${env.EC2_HOST}:${env.DEPLOY_DIR}/
                        
                        echo "=== Stopping old Eureka instance (if any) ==="
                        ssh -i ${env.PEM_PATH} -o StrictHostKeyChecking=no ec2-user@${env.EC2_HOST} "pkill -f ${env.SERVICE_NAME}.jar || true"
                        
                        echo "=== Starting new Eureka service ==="
                        ssh -i ${env.PEM_PATH} -o StrictHostKeyChecking=no ec2-user@${env.EC2_HOST} "nohup java -jar ${env.DEPLOY_DIR}/${env.SERVICE_NAME}.jar --server.port=8761 > ${env.DEPLOY_DIR}/logs/eureka.log 2>&1 &"
                        
                        echo "=== Deployment complete ==="
                    '
                """
            }
        }
    }

    post {
        success {
            echo "✅ Eureka deployed successfully to EC2 (${env.EC2_HOST})!"
        }
        failure {
            echo "❌ Deployment failed. Check Jenkins console logs for details."
        }
    }
}
