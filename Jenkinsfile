pipeline {
    agent any

    tools {
        jdk 'Java17'
        maven 'Maven3'
    }

    environment {
        EC2_USER = "ec2-user"
        EC2_HOST = "13.60.47.188"
        PEM_PATH = "C:\\Users\\KRISHNA\\Downloads\\krishna.pem"
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
                bat '''
                    echo ===== Copying JAR to EC2 =====
                    scp -i "%PEM_PATH%" -o StrictHostKeyChecking=no target\\%SERVICE_NAME%.jar %EC2_USER%@%EC2_HOST%:/home/%EC2_USER%/

                    echo ===== Stopping old Eureka instance if running =====
                    ssh -i "%PEM_PATH%" -o StrictHostKeyChecking=no %EC2_USER%@%EC2_HOST% "pkill -f %SERVICE_NAME%.jar || true"

                    echo ===== Starting new Eureka instance =====
                    ssh -i "%PEM_PATH%" -o StrictHostKeyChecking=no %EC2_USER%@%EC2_HOST% "nohup java -jar /home/%EC2_USER%/%SERVICE_NAME%.jar > eureka.log 2>&1 &"
                '''
            }
        }
    }

    post {
        success {
            echo '✅ Eureka deployed successfully to EC2!'
        }
        failure {
            echo '❌ Deployment failed. Check Jenkins console logs for details.'
        }
    }
}
