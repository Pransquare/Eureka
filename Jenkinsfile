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

        stage('Deploy to EC2 via WinRM and S3') {
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
                
                # WinRM session options
                \$sessOption = New-PSSessionOption -SkipCACheck -SkipCNCheck -SkipRevocationCheck
                
                # Create WinRM session
                \$session = New-PSSession -ComputerName '13.53.193.215' -UseSSL -Credential \$cred -Authentication Basic -SessionOption \$sessOption
                
                # On EC2, download JAR from S3 using AWS CLI
                Invoke-Command -Session \$session -ScriptBlock {
                    # Make sure AWS CLI is installed and configured
                    if (-Not (Test-Path 'C:\\Deployments')) {
                        New-Item -ItemType Directory -Path 'C:\\Deployments'
                    }

                    # Download from S3
                    aws s3 cp s3://eureka-deployment-2025/eureka-server.jar C:\\Deployments\\eureka-server.jar

                    # Stop existing Java process (if needed)
                    Get-Process -Name java -ErrorAction SilentlyContinue | Stop-Process -Force

                    # Run the JAR
                    Start-Process -FilePath 'java' -ArgumentList '-jar C:\\Deployments\\eureka-server.jar' -WindowStyle Hidden
                }

                # Close session
                Remove-PSSession \$session
            """
        }
    }
}

    }
}
