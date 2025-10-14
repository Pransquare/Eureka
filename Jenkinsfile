pipeline {
    agent any

    tools {
        jdk 'Java17'
        maven 'Maven3'
    }

    environment {
        DEPLOY_DIR = "/opt/eureka"
        EC2_HOST = "13.60.47.188"
        SERVICE_NAME = "eureka-0.0.1-SNAPSHOT.jar"
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
                sshagent(['EC2_SSH_CREDENTIAL_ID']) {
                    bat '''
                        echo ===== Deploying to EC2 =====
                        "C:\\Program Files\\Git\\bin\\bash.exe" -c "
                            ssh -o StrictHostKeyChecking=no ec2-user@13.60.47.188 'mkdir -p /opt/eureka/logs';
                            scp -o StrictHostKeyChecking=no target/eureka-0.0.1-SNAPSHOT.jar ec2-user@13.60.47.188:/opt/eureka/;
                            ssh -o StrictHostKeyChecking=no ec2-user@13.60.47.188 'pkill -f eureka-0.0.1-SNAPSHOT.jar || true';
                            ssh -o StrictHostKeyChecking=no ec2-user@13.60.47.188 'nohup java -jar /opt/eureka/eureka-0.0.1-SNAPSHOT.jar --server.port=8761 > /opt/eureka/logs/eureka.log 2>&1 &';
                        "
                    '''
                }
            }
        }
    }

    post {
        success {
            echo '✅ Eureka deployed successfully to EC2!'
        }
        failure {
            echo '❌ Deployment failed. Check console output for details.'
        }
    }
}
