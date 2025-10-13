pipeline {
    agent any
    tools {
        jdk 'Java17'
        maven 'Maven3'
    }

    environment {
        DEPLOY_DIR = "C:\\Deployments"
        SERVICE_NAME = "eureka-server"
        S3_BUCKET = "eureka-deployment-2025"
        REGION = "eu-north-1"
        EC2_HOST = "13.53.193.215"
    }

    stages {
        stage('Checkout') {
            steps {
                git branch: 'master', url: 'https://github.com/Pransquare/Eureka.git'
            }
        }

        stage('Build') {
            steps {
                bat 'mvn clean package -DskipTests'
            }
        }

        stage('Upload JAR to S3') {
            steps {
                withAWS(credentials: 'aws-jenkins-creds', region: "${REGION}") {
                    s3Upload(bucket: "${S3_BUCKET}", path: "${SERVICE_NAME}.jar", file: "target\\${SERVICE_NAME}.jar")
                }
            }
        }

        stage('Deploy to EC2 via WinRM') {
            steps {
                withCredentials([usernamePassword(
                    credentialsId: 'ec2-admin-creds',
                    usernameVariable: 'EC2_USER',
                    passwordVariable: 'EC2_PASSWORD'
                )]) {
                    powershell """
                        # Convert password to secure string
                        \$secPassword = ConvertTo-SecureString '\$EC2_PASSWORD' -AsPlainText -Force
                        \$cred = New-Object System.Management.Automation.PSCredential('\$EC2_USER', \$secPassword)
                        
                        # Create WinRM session
                        \$session = New-PSSession -ComputerName '${EC2_HOST}' -Credential \$cred -Authentication Basic

                        # Run commands on EC2
                        Invoke-Command -Session \$session -ScriptBlock {
                            if (-Not (Test-Path '${DEPLOY_DIR}')) {
                                New-Item -ItemType Directory -Path '${DEPLOY_DIR}'
                            }

                            # Download JAR from S3
                            aws s3 cp s3://${S3_BUCKET}/${SERVICE_NAME}.jar ${DEPLOY_DIR}\\${SERVICE_NAME}.jar

                            # Stop existing Java process
                            Get-Process -Name java -ErrorAction SilentlyContinue | Stop-Process -Force

                            # Start the JAR
                            Start-Process -FilePath 'java' -ArgumentList "-jar ${DEPLOY_DIR}\\${SERVICE_NAME}.jar" -WindowStyle Hidden
                        }

                        # Remove session
                        Remove-PSSession \$session
                    """
                }
            }
        }
    }
}
