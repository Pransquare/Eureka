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

        stage('Deploy to EC2 via WinRM HTTPS') {
            steps {
                withCredentials([usernamePassword(
                    credentialsId: 'ec2-admin-creds',
                    usernameVariable: 'EC2_USER',
                    passwordVariable: 'EC2_PASSWORD'
                )]) {
                    powershell """
                        # Convert the password to a secure string
                        \$secPassword = ConvertTo-SecureString '\$EC2_PASSWORD' -AsPlainText -Force
                        
                        # Create PSCredential object
                        \$cred = New-Object System.Management.Automation.PSCredential('\$EC2_USER', \$secPassword)
                        
                        # Define WinRM session options
                        \$sessOption = New-PSSessionOption -SkipCACheck -SkipCNCheck -SkipRevocationCheck
                        
                        # Connect to EC2 via WinRM HTTPS
                        \$session = New-PSSession -ComputerName '13.53.193.215' -UseSSL -Credential \$cred -Authentication Basic -SessionOption \$sessOption
                        
                        # Run deployment commands on EC2
                        Invoke-Command -Session \$session -ScriptBlock {
                            param(\$bucket, \$serviceName, \$deployDir)
                            
                            # Ensure deployment directory exists
                            if (-not (Test-Path \$deployDir)) { New-Item -ItemType Directory -Path \$deployDir }
                            
                            # Download JAR from S3
                            aws s3 cp s3://\$bucket/\$serviceName.jar "\$deployDir\\\$serviceName.jar" --region ${REGION}
                            
                            # Stop existing Java process if running
                            \$proc = Get-Process -Name "java" -ErrorAction SilentlyContinue
                            if (\$proc) { Stop-Process -Name "java" -Force }
                            
                            # Start the new JAR
                            Start-Process -FilePath "java" -ArgumentList "-jar \$deployDir\\\$serviceName.jar" -NoNewWindow
                        } -ArgumentList "${S3_BUCKET}", "${SERVICE_NAME}", "${DEPLOY_DIR}"
                        
                        # Close the session
                        Remove-PSSession \$session
                    """
                }
            }
        }
    }
}
