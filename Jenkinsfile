pipeline {
    agent any

    tools {
        jdk 'Java17'      // Must match JDK name in Jenkins Global Tool Configuration
        maven 'Maven3'    // Must match Maven name in Jenkins Global Tool Configuration
    }

    environment {
        EC2_USER = 'ec2-user'
        EC2_HOST = '13.53.39.170'        // Replace with your EC2 public IP
        DEPLOY_DIR = '/home/ec2-user/services/eureka-server'
        APP_NAME = 'eureka-server'
        JAR_NAME = 'eureka-server.jar'
    }

    stages {

        stage('Checkout SCM') {
            steps {
                echo '===== Cloning repository from GitHub ====='
                git branch: 'master', url: 'https://github.com/Pransquare/Eureka.git'
            }
        }

        stage('Verify Java & Maven') {
            steps {
                echo '===== Verifying Java and Maven ====='
                bat 'java -version'
                bat 'echo %JAVA_HOME%'
                bat 'mvn -version'
            }
        }

        stage('Build') {
            steps {
                echo '===== Building project using Maven ====='
                bat "mvn clean package -DskipTests"
            }
        }

        stage('Prepare Deploy Script') {
            steps {
                echo '===== Creating deploy.sh script ====='
                script {
                    writeFile file: 'deploy.sh', text: """
                        #!/bin/bash
                        mkdir -p ${DEPLOY_DIR}
                        cd ${DEPLOY_DIR}

                        # Stop previous instance if running
                        PID=\$(pgrep -f ${JAR_NAME})
                        if [ ! -z "\$PID" ]; then
                            echo "Stopping previous Eureka instance (PID: \$PID)"
                            kill -9 \$PID
                        fi

                        # Copy new jar
                        cp ${WORKSPACE}/target/${JAR_NAME} ${DEPLOY_DIR}/

                        # Run in background
                        nohup java -jar ${DEPLOY_DIR}/${JAR_NAME} > ${DEPLOY_DIR}/eureka.log 2>&1 &
                        echo "Eureka server deployed and started successfully!"
                    """
                }
            }
        }

        stage('Deploy to EC2') {
            steps {
                echo '===== Deploying application to EC2 ====='
                sshPublisher(publishers: [
                    sshPublisherDesc(
                        configName: 'ec2-server',   // Must be configured in Jenkins SSH credentials
                        transfers: [
                            sshTransfer(
                                sourceFiles: 'deploy.sh,target/' + JAR_NAME,
                                remoteDirectory: DEPLOY_DIR,
                                removePrefix: '',
                                execCommand: """
                                    chmod +x ${DEPLOY_DIR}/deploy.sh
                                    ${DEPLOY_DIR}/deploy.sh
                                """
                            )
                        ],
                        verbose: true
                    )
                ])
            }
        }
    }

    post {
        success {
            echo 'Deployment completed successfully!'
        }
        failure {
            echo 'Deployment failed.'
        }
    }
}
