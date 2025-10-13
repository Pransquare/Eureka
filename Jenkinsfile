pipeline {
    agent any
    tools {
        jdk 'Java17'
        maven 'Maven3'
    }

    environment {
        DEPLOY_DIR = "C:\\Deployments"
        SERVICE_NAME = "eureka-server"
        SERVICE_PORT = "8761"
        S3_BUCKET = "eureka-deployment-2025"
        REGION = "eu-north-1"
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
                powershell """
                    # Hardcoded credentials (temporary for testing)
                    \$username = 'Administrator'
                    \$password = 'd8%55Ir.%Z!hNR%VgUe-07OYX0ujLy;S'
                    \$secPassword = ConvertTo-SecureString \$password -AsPlainText -Force
                    \$cred = New-Object System.Management.Automation.PSCredential(\$username, \$secPassword)

                    # Session options to skip certificate checks
                    \$sessOption = New-PSSessionOption -SkipCACheck -SkipCNCheck -SkipRevocationCheck

                    # Create WinRM session using HTTPS
                    \$session = New-PSSession -ComputerName '13.53.193.215' -UseSSL -Credential \$cred -Authentication Basic -SessionOption \$sessOption

                    # Commands to deploy JAR
                    Invoke-Command -Session \$session -ScriptBlock {
                        if (-Not (Test-Path '${DEPLOY_DIR}')) {
                            New-Item -ItemType Directory -Path '${DEPLOY_DIR}'
                        }

                        # Download JAR from S3
                        aws s3 cp s3://${S3_BUCKET}/${SERVICE_NAME}.jar ${DEPLOY_DIR}\\${SERVICE_NAME}.jar

                        # Stop existing Java process
                        Get-Process -Name java -ErrorAction SilentlyContinue | Stop-Process -Force

                        # Start the new JAR
                        Start-Process -FilePath 'java' -ArgumentList "-jar ${DEPLOY_DIR}\\${SERVICE_NAME}.jar" -WindowStyle Hidden
                    }

                    # Close session
                    Remove-PSSession \$session
                """
            }
        }
    }
}
