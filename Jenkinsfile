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
        EC2_HOST = "13.53.193.215" // Replace with your EC2 public IP
        EC2_USER = "Administrator"
        EC2_PASS = "d8%55Ir.%Z!hNR%VgUe-07OYX0ujLy;S"
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
                    # Convert password to secure string
                    \$secPassword = ConvertTo-SecureString '${EC2_PASS}' -AsPlainText -Force
                    \$cred = New-Object System.Management.Automation.PSCredential('${EC2_USER}', \$secPassword)

                    # Session options to skip certificate checks
                    \$sessOption = New-PSSessionOption -SkipCACheck -SkipCNCheck -SkipRevocationCheck

                    # Create WinRM session using HTTPS
                    \$session = New-PSSession -ComputerName '${EC2_HOST}' -UseSSL -Credential \$cred -Authentication Basic -SessionOption \$sessOption

                    # Deploy commands
                    Invoke-Command -Session \$session -ScriptBlock {
                        # Create deployment directory if missing
                        if (-Not (Test-Path '${DEPLOY_DIR}')) {
                            New-Item -ItemType Directory -Path '${DEPLOY_DIR}'
                        }

                        # Download JAR from S3
                        aws s3 cp s3://${S3_BUCKET}/${SERVICE_NAME}.jar ${DEPLOY_DIR}\\${SERVICE_NAME}.jar

                        # Stop existing JAR process safely
                        $oldProcess = Get-Process -Name java -ErrorAction SilentlyContinue | Where-Object { $_.Path -like "*${SERVICE_NAME}.jar*" }
                        if ($oldProcess) { $oldProcess | Stop-Process -Force }

                        # Optional wait
                        Start-Sleep -Seconds 3

                        # Start the new JAR and log output
                        Start-Process -FilePath 'java' -ArgumentList "-jar ${DEPLOY_DIR}\\${SERVICE_NAME}.jar > ${DEPLOY_DIR}\\logs.txt 2>&1" -WindowStyle Hidden
                    }

                    # Close session
                    Remove-PSSession \$session
                """
            }
        }
    }
}
