pipeline {
    agent any
    tools {
        jdk 'Java17'
        maven 'Maven3'
    }

    environment {
        DEPLOY_DIR = "C:\\Apps\\eureka-server"
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
                
                # Define WinRM session options (adjust as needed)
                \$sessOption = New-PSSessionOption -SkipCACheck -SkipCNCheck -SkipRevocationCheck
                
                # Connect to EC2 via WinRM HTTPS
                \$session = New-PSSession -ComputerName 'YOUR_EC2_PUBLIC_IP' -UseSSL -Credential \$cred -Authentication Basic -SessionOption \$sessOption
                
                # Copy the JAR to remote EC2
                Copy-Item -Path 'C:\\ProgramData\\Jenkins\\.jenkins\\workspace\\Eureka-deploy-AWS\\target\\eureka-server.jar' -Destination 'C:\\Deployments\\eureka-server.jar' -ToSession \$session
                
                # Run deployment command on EC2
                Invoke-Command -Session \$session -ScriptBlock { java -jar C:\\Deployments\\eureka-server.jar }
                
                # Close the session
                Remove-PSSession \$session
            """
        }
    }
}

    }
}
