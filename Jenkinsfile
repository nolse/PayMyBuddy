def deployToServer(String hostname, String sshCredential) {
    sh 'apk --no-cache add openssh-client curl'

    sshagent(credentials: [sshCredential]) {
        sh """
            set -e

            mkdir -p ~/.ssh
            chmod 700 ~/.ssh
            ssh-keyscan -T 30 -t rsa ${hostname} >> ~/.ssh/known_hosts

            scp src/main/resources/database/create.sql ubuntu@${hostname}:/home/ubuntu/create.sql

            ssh ubuntu@${hostname} "
                set -e

                if ! command -v docker >/dev/null 2>&1; then
                    curl -fsSL https://get.docker.com -o install-docker.sh
                    sudo sh install-docker.sh
                    sudo usermod -aG docker ubuntu
                fi

                echo \$DOCKERHUB_AUTH_PSW | docker login -u \$DOCKERHUB_AUTH_USR --password-stdin

                docker pull \$ID_DOCKER/\$IMAGE_NAME:\$IMAGE_TAG

                docker rm -f paymybuddy mysql-db || true
                docker network rm app-network || true
                docker volume create mysql-data || true
                docker network create app-network

                docker run -d --name mysql-db \\
                    --network app-network \\
                    -e MYSQL_ROOT_PASSWORD=\$MYSQL_ROOT_PASSWORD \\
                    -v mysql-data:/var/lib/mysql \\
                    -v /home/ubuntu/create.sql:/docker-entrypoint-initdb.d/create.sql \\
                    mysql:8.0

                echo 'Waiting for MySQL...'
                until docker exec mysql-db mysqladmin ping -h localhost --silent; do
                    sleep 5
                done

                docker run -d -p 80:8080 --name paymybuddy \\
                    --network app-network \\
                    -e SPRING_DATASOURCE_URL=jdbc:mysql://mysql-db:3306/db_paymybuddy \\
                    -e SPRING_DATASOURCE_USERNAME=root \\
                    -e SPRING_DATASOURCE_PASSWORD=\$MYSQL_ROOT_PASSWORD \\
                    \$ID_DOCKER/\$IMAGE_NAME:\$IMAGE_TAG
            "
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
        HOSTNAME_DEPLOY_STAGING = "13.220.129.1"
        HOSTNAME_DEPLOY_PROD    = "34.205.255.65"
        SONAR_TOKEN             = credentials('jenkins-sonar')
        MYSQL_ROOT_PASSWORD     = credentials('MYSQL_ROOT_PASSWORD')
    }

    stages {

        stage('Clean Workspace') {
            agent any
            steps {
                cleanWs()
            }
        }

        stage('Tests') {
            agent {
                docker {
                    image 'maven:3.8.6-amazoncorretto-17'
                    args '-v /root/.m2:/root/.m2'
                }
            }
            steps {
                sh 'mvn clean test'
            }
        }

        stage('Code Quality') {
            agent {
                docker {
                    image 'maven:3.8.6-amazoncorretto-17'
                    args '-v /root/.m2:/root/.m2'
                }
            }
            steps {
                sh """
                    mvn sonar:sonar \
                      -Dsonar.projectKey=spring-boot-app \
                      -Dsonar.organization=alpha-jenkins \
                      -Dsonar.host.url=https://sonarcloud.io \
                      -Dsonar.login=${SONAR_TOKEN}
                """
            }
        }

        stage("Quality Gate") {
            agent any
            steps {
                timeout(time: 2, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: true
                }
            }
        }

        stage('Build') {
            agent {
                docker {
                    image 'maven:3.8.6-amazoncorretto-17'
                    args '-v /root/.m2:/root/.m2'
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
                sh """
                    docker build -t ${ID_DOCKER}/${IMAGE_NAME}:${IMAGE_TAG} .
                    echo \$DOCKERHUB_AUTH_PSW | docker login -u \$DOCKERHUB_AUTH_USR --password-stdin
                    docker push ${ID_DOCKER}/${IMAGE_NAME}:${IMAGE_TAG}
                """
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
                sh """
                    apk --no-cache add curl
                    sleep 15
                    STATUS=\$(curl -s -o /dev/null -w "%{http_code}" http://${HOSTNAME_DEPLOY_STAGING})
                    echo "HTTP Status: \$STATUS"
                    [ "\$STATUS" = "200" ] || [ "\$STATUS" = "302" ]
                """
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
                sh """
                    apk --no-cache add curl
                    sleep 15
                    STATUS=\$(curl -s -o /dev/null -w "%{http_code}" http://${HOSTNAME_DEPLOY_PROD})
                    echo "HTTP Status: \$STATUS"
                    [ "\$STATUS" = "200" ] || [ "\$STATUS" = "302" ]
                """
            }
        }
    }

    post {
        success {
            slackSend(
                channel: '#jenkins-eazytraining-alpha-alerte',
                color: '#00FF00',
                message: "SUCCESS: ${env.JOB_NAME} [${env.BUILD_NUMBER}] ${env.BUILD_URL}"
            )
        }
        failure {
            slackSend(
                channel: '#jenkins-eazytraining-alpha-alerte',
                color: '#FF0000',
                message: "FAILED: ${env.JOB_NAME} [${env.BUILD_NUMBER}] ${env.BUILD_URL}"
            )
        }
        unstable {
            slackSend(
                channel: '#jenkins-eazytraining-alpha-alerte',
                color: '#FFA500',
                message: "UNSTABLE: ${env.JOB_NAME} [${env.BUILD_NUMBER}] ${env.BUILD_URL}"
            )
        }
    }
}
