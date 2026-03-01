def deployToServer(String hostname, String sshCredential) {
    sh 'apk --no-cache add openssh-client'
    sshagent(credentials: [sshCredential]) {
        sh """
            [ -d ~/.ssh ] || mkdir ~/.ssh && chmod 0700 ~/.ssh
            ssh-keyscan -t rsa ${hostname} >> ~/.ssh/known_hosts

            ssh -t ubuntu@${hostname} "
                if ! command -v docker &> /dev/null; then
                    curl -fsSL https://get.docker.com -o install-docker.sh
                    sudo sh install-docker.sh
                    sudo usermod -aG docker ubuntu
                fi
            "

            command1="docker login -u \$DOCKERHUB_AUTH_USR -p \$DOCKERHUB_AUTH_PSW"
            command2="docker pull \$ID_DOCKER/\$IMAGE_NAME:\$IMAGE_TAG"
            command3="docker rm -f paymybuddy mysql-db || echo 'containers do not exist'"
            command4="docker network rm app-network || true"
            command5="docker network create app-network"
            command6="docker run -d --name mysql-db --network app-network \
                -e MYSQL_ROOT_PASSWORD=\$MYSQL_ROOT_PASSWORD \
                -e MYSQL_DATABASE=db_paymybuddy \
                -v /home/ubuntu/sql:/docker-entrypoint-initdb.d \
                   mysql:8.0"
            command7="sleep 30"
            command8="docker run -d -p 80:8080 --name paymybuddy --network app-network \\
                -e SPRING_DATASOURCE_URL=jdbc:mysql://mysql-db:3306/db_paymybuddy \\
                -e SPRING_DATASOURCE_USERNAME=root \\
                -e SPRING_DATASOURCE_PASSWORD=\$MYSQL_ROOT_PASSWORD \\
                \$ID_DOCKER/\$IMAGE_NAME:\$IMAGE_TAG"

            ssh -t ubuntu@${hostname} \\
                "\$command1 && \$command2 && \$command3 && \$command4 && \$command5 && \$command6 && \$command7 && \$command8"
        """
    }
}

pipeline {
    agent none
    environment {
        DOCKERHUB_AUTH          = credentials('DOCKERHUB_AUTH')
        ID_DOCKER               = "${DOCKERHUB_AUTH_USR}"
        IMAGE_NAME              = "paymybuddy"
        IMAGE_TAG               = "latest"
        HOSTNAME_DEPLOY_STAGING = "100.48.221.157"
        HOSTNAME_DEPLOY_PROD    = "3.83.78.46"
        SONAR_TOKEN             = credentials('jenkins-sonar')
        MYSQL_ROOT_PASSWORD     = credentials('MYSQL_ROOT_PASSWORD')
    }

    stages {

        stage('Tests') {
            agent {
                docker {
                    image 'maven:3.8.6-amazoncorretto-17'
                    args  '-u root -v /root/.m2:/root/.m2'
                }
            }
            steps {
                sh 'mvn test'
            }
        }

        stage('Code Quality') {
            agent {
                docker {
                    image 'maven:3.8.6-amazoncorretto-17'
                    args  '-u root -v /root/.m2:/root/.m2'
                }
            }
            steps {
                sh '''
                    mvn sonar:sonar \
                        -Dsonar.projectKey=spring-boot-app \
                        -Dsonar.organization=alpha-jenkins \
                        -Dsonar.host.url=https://sonarcloud.io \
                        -Dsonar.login=${SONAR_TOKEN}
                '''
            }
        }

        stage('Build') {
            agent {
                docker {
                    image 'maven:3.8.6-amazoncorretto-17'
                    args  '-u root -v /root/.m2:/root/.m2'
                }
            }
            steps {
                sh 'mvn package -DskipTests'
            }
        }

        stage('Build & Push Docker Image') {
            agent {
                docker {
                    image 'docker:latest'
                    args '-u root -v /var/run/docker.sock:/var/run/docker.sock'
                }
            }
            steps {
                sh 'docker build -t ${ID_DOCKER}/${IMAGE_NAME}:${IMAGE_TAG} .'
                sh '''
                    docker login -u $DOCKERHUB_AUTH_USR -p $DOCKERHUB_AUTH_PSW
                    docker push ${ID_DOCKER}/${IMAGE_NAME}:${IMAGE_TAG}
                '''
            }
        }

        stage('Deploy Staging') {
            when { branch 'main' }
            agent {
                docker {
                    image 'alpine'
                    args '-u root'
                }
            }
            steps {
                script {
                    deployToServer(env.HOSTNAME_DEPLOY_STAGING, 'SSH_AUTH_STAGING')
                }
            }
        }

        stage('Test Staging') {
            when { branch 'main' }
            agent {
                docker {
                    image 'alpine'
                    args '-u root'
                }
            }
            steps {
                sh '''
                    apk --no-cache add curl
                    sleep 10
                    STATUS=$(curl -s -o /dev/null -w "%{http_code}" http://${HOSTNAME_DEPLOY_STAGING})
                    echo "HTTP Status: $STATUS"
                    if [ "$STATUS" = "200" ] || [ "$STATUS" = "302" ]; then
                        echo "Application is UP!"
                    else
                        echo "Application is DOWN! Status: $STATUS"
                        exit 1
                    fi
                '''
            }
        }

        stage('Deploy Prod') {
            when { branch 'main' }
            agent {
                docker {
                    image 'alpine'
                    args '-u root'
                }
            }
            steps {
                script {
                    deployToServer(env.HOSTNAME_DEPLOY_PROD, 'SSH_AUTH_PROD')
                }
            }
        }

        stage('Test Prod') {
            when { branch 'main' }
            agent {
                docker {
                    image 'alpine'
                    args '-u root'
                }
            }
            steps {
                sh '''
                    apk --no-cache add curl
                    sleep 10
                    STATUS=$(curl -s -o /dev/null -w "%{http_code}" http://${HOSTNAME_DEPLOY_PROD})
                    echo "HTTP Status: $STATUS"
                    if [ "$STATUS" = "200" ] || [ "$STATUS" = "302" ]; then
                        echo "Application is UP!"
                    else
                        echo "Application is DOWN! Status: $STATUS"
                        exit 1
                    fi
                '''
            }
        }

    }

    post {
        success {
            slackSend(
                channel: '#jenkins-eazytraining-alpha-alerte',
                color: '#00FF00',
                message: "✅ SUCCESSFUL: Job '${env.JOB_NAME} [${env.BUILD_NUMBER}]' (${env.BUILD_URL})"
            )
        }
        failure {
            slackSend(
                channel: '#jenkins-eazytraining-alpha-alerte',
                color: '#FF0000',
                message: "❌ FAILED: Job '${env.JOB_NAME} [${env.BUILD_NUMBER}]' (${env.BUILD_URL})"
            )
        }
        unstable {
            slackSend(
                channel: '#jenkins-eazytraining-alpha-alerte',
                color: '#FFA500',
                message: "⚠️ UNSTABLE: Job '${env.JOB_NAME} [${env.BUILD_NUMBER}]' (${env.BUILD_URL})"
            )
        }
    }
}
