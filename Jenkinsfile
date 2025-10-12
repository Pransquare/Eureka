pipeline {
    agent any

    tools {
        jdk 'Java17'
        maven 'Maven3'
    }

    environment {
        EC2_HOST = "13.53.193.215"
        DEPLOY_DIR = "C:\\Apps\\eureka-server"
        SERVICE_NAME = "eureka-server"
        SERVICE_PORT = "8761"
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

        stage('Deploy to EC2') {
            steps {
                // Use Jenkins credentials for secure handling
                withCredentials([usernamePassword(credentialsId: 'ec2-admin-creds',
                                                  usernameVariable: 'EC2_USER',
                                                  passwordVariable: 'EC2_PASSWORD')]) {
                    script {
                        powershell """
                        # Convert password to secure string
                        \$secpasswd = ConvertTo-SecureString '${EC2_PASSWORD}' -AsPlainText -Force
                        \$cred = New-Object System.Management.Automation.PSCredential ('${EC2_USER}', \$secpasswd)

                        # 1️⃣ Create deployment directory if it doesn't exist
                        Invoke-Command -ComputerName ${EC2_HOST} -Credential \$cred -UseSSL -ScriptBlock {
                            if (-not (Test-Path -Path '${DEPLOY_DIR}')) {
                                New-Item -Path '${DEPLOY_DIR}' -ItemType Directory | Out-Null
                            }
                        }

                        # 2️⃣ Stop existing Java processes
                        Invoke-Command -ComputerName ${EC2_HOST} -Credential \$cred -UseSSL -ScriptBlock {
                            Get-Process java -ErrorAction SilentlyContinue | Stop-Process -Force
                        }

                        # 3️⃣ Copy the JAR to EC2 (over administrative share or S3 download)
                        # Here we use SMB share (ensure Jenkins IP has access) - you can switch to S3 if needed
                        Copy-Item -Path 'target\\${SERVICE_NAME}.jar' -Destination "\\\\${EC2_HOST}\\C\$\\Apps\\eureka-server\\" -Credential \$cred -Force

                        # 4️⃣ Start the Spring Boot service
                        Invoke-Command -ComputerName ${EC2_HOST} -Credential \$cred -UseSSL -ScriptBlock {
                            Start-Process -FilePath 'java' -ArgumentList "-jar ${DEPLOY_DIR}\\${SERVICE_NAME}.jar --server.port=${SERVICE_PORT}" -NoNewWindow
                        }
                        """
                    }
                }
            }
        }
    }
}
