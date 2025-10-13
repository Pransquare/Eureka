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
        EC2_HOST = "13.53.193.215" // Change as needed
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

        stage('Deploy to EC2 via WinRM HTTPS') {
            steps {
                withCredentials([usernamePassword(
                    credentialsId: 'ec2-admin-creds', 
                    usernameVariable: 'EC2_USER', 
                    passwordVariable: 'EC2_PASSWORD'
                )]) {
                    powershell """
                        # Convert password to secure string
                        \$secPassword = ConvertTo-SecureString "\$env:EC2_PASSWORD" -AsPlainText -Force
                        
                        # Create PSCredential object
                        \$cred = New-Object System.Management.Automation.PSCredential("\$env:EC2_USER", \$secPassword)
                        
                        # WinRM session options to skip SSL checks
                        \$sessOption = New-PSSessionOption -SkipCACheck -SkipCNCheck -SkipRevocationCheck
                        
                        # Create a WinRM session to EC2
                        \$session = New-PSSession -ComputerName "${EC2_HOST}" -UseSSL -Credential \$cred -Authentication Basic -SessionOption \$sessOption
                        
                        # Ensure deployment directory exists
                        Invoke-Command -Session \$session -ScriptBlock { param(\$dir) if (-Not (Test-Path \$dir)) { New-Item -ItemType Directory -Path \$dir } } -ArgumentList "${DEPLOY_DIR}"
                        
                        # Copy JAR to EC2
                        Copy-Item -Path "C:\\ProgramData\\Jenkins\\.jenkins\\workspace\\Eureka-deploy-AWS\\target\\${SERVICE_NAME}.jar" -Destination "${DEPLOY_DIR}\\${SERVICE_NAME}.jar" -ToSession \$session
                        
                        # Stop any running instance (optional)
                        Invoke-Command -Session \$session -ScriptBlock { 
                            \$proc = Get-Process -Name "java" -ErrorAction SilentlyContinue
                            if (\$proc) { Stop-Process -Name "java" -Force }
                        }

                        # Start new JAR
                        Invoke-Command -Session \$session -ScriptBlock { Start-Process -FilePath "java" -ArgumentList "-jar ${DEPLOY_DIR}\\${SERVICE_NAME}.jar" -NoNewWindow }
                        
                        # Close session
                        Remove-PSSession \$session
                    """
                }
            }
        }
    }
}
