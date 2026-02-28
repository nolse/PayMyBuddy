pipeline {
    agent none
    environment {
        DOCKERHUB_AUTH      = credentials('DOCKERHUB_AUTH')
        ID_DOCKER           = "${DOCKERHUB_AUTH_USR}"
        IMAGE_NAME          = "paymybuddy"
        IMAGE_TAG           = "latest"
        HOSTNAME_DEPLOY_STAGING = "13.220.129.1"
        HOSTNAME_DEPLOY_PROD    = "34.205.255.65"
        SONAR_TOKEN         = credentials('jenkins-sonar')
    }

    stages {

        stage('Tests') {
            agent {
                docker {
                    image 'maven:3.8.6-amazoncorretto-17'
                    args  '-v /root/.m2:/root/.m2'
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
                    args  '-v /root/.m2:/root/.m2'
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
            args '-v /root/.m2:/root/.m2'
        }
    }
    steps {
        sh 'mvn package -DskipTests'
    }
}

stage('Build & Push Docker Image') {
    agent any
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
            agent any
            steps {
                sshagent(credentials: ['SSH_AUTH_STAGING']) {
                    sh '''
                        [ -d ~/.ssh ] || mkdir ~/.ssh && chmod 0700 ~/.ssh
                        ssh-keyscan -t rsa ${HOSTNAME_DEPLOY_STAGING} >> ~/.ssh/known_hosts

                        ssh -t ubuntu@${HOSTNAME_DEPLOY_STAGING} "
                            if ! command -v docker &> /dev/null; then
                                curl -fsSL https://get.docker.com -o install-docker.sh
                                sudo sh install-docker.sh
                                sudo usermod -aG docker ubuntu
                            fi
                        "

                        command1="docker login -u $DOCKERHUB_AUTH_USR -p $DOCKERHUB_AUTH_PSW"
                        command2="docker pull $ID_DOCKER/$IMAGE_NAME:$IMAGE_TAG"
                        command3="docker rm -f webapp mysql-db || echo 'containers do not exist'"
                        command4="docker network create app-network || true"
                        command5="docker run -d --name mysql-db --network app-network \
                            -e MYSQL_ROOT_PASSWORD=password \
                            -e MYSQL_DATABASE=db_paymybuddy mysql:8.0"
                        command6="sleep 60"
                        command7="docker run -d -p 80:8080 --name webapp --network app-network \
                            -e SPRING_DATASOURCE_URL=jdbc:mysql://mysql-db:3306/db_paymybuddy \
                            -e SPRING_DATASOURCE_USERNAME=root \
                            -e SPRING_DATASOURCE_PASSWORD=password \
                            $ID_DOCKER/$IMAGE_NAME:$IMAGE_TAG"

                        ssh -t ubuntu@${HOSTNAME_DEPLOY_STAGING} \
                            "$command1 && $command2 && $command3 && $command4 && $command5 && $command6 && $command7"
                    '''
                }
            }
        }

        stage('Test Staging') {
            when { branch 'main' }
            agent {
                docker {
                    image 'alpine'
                }
            }
            steps {
                sh '''
                    apk --no-cache add curl
                    sleep 60
                    curl -I http://${HOSTNAME_DEPLOY_STAGING} | grep -q "200"
                '''
            }
        }

        stage('Deploy Prod') {
            when { branch 'main' }
            agent any
            steps {
                sshagent(credentials: ['SSH_AUTH_PROD']) {
                    sh '''
                        [ -d ~/.ssh ] || mkdir ~/.ssh && chmod 0700 ~/.ssh
                        ssh-keyscan -t rsa ${HOSTNAME_DEPLOY_PROD} >> ~/.ssh/known_hosts

                        ssh -t ubuntu@${HOSTNAME_DEPLOY_PROD} "
                            if ! command -v docker &> /dev/null; then
                                curl -fsSL https://get.docker.com -o install-docker.sh
                                sudo sh install-docker.sh
                                sudo usermod -aG docker ubuntu
                            fi
                        "

                        command1="docker login -u $DOCKERHUB_AUTH_USR -p $DOCKERHUB_AUTH_PSW"
                        command2="docker pull $ID_DOCKER/$IMAGE_NAME:$IMAGE_TAG"
                        command3="docker rm -f webapp mysql-db || echo 'containers do not exist'"
                        command4="docker network create app-network || true"
                        command5="docker run -d --name mysql-db --network app-network \
                            -e MYSQL_ROOT_PASSWORD=password \
                            -e MYSQL_DATABASE=db_paymybuddy mysql:8.0"
                        command6="sleep 60"
                        command7="docker run -d -p 80:8080 --name webapp --network app-network \
                            -e SPRING_DATASOURCE_URL=jdbc:mysql://mysql-db:3306/db_paymybuddy \
                            -e SPRING_DATASOURCE_USERNAME=root \
                            -e SPRING_DATASOURCE_PASSWORD=password \
                            $ID_DOCKER/$IMAGE_NAME:$IMAGE_TAG"

                        ssh -t ubuntu@${HOSTNAME_DEPLOY_PROD} \
                            "$command1 && $command2 && $command3 && $command4 && $command5 && $command6 && $command7"
                    '''
                }
            }
        }

        stage('Test Prod') {
            when { branch 'main' }
            agent {
                docker {
                    image 'alpine'
                }
            }
            steps {
                sh '''
                    apk --no-cache add curl
                    sleep 60
                    curl -I http://${HOSTNAME_DEPLOY_PROD} | grep -q "200"
                '''
            }
        }

    }

post {
    success {
        slackSend(
            channel: '#jenkins-eazytraining-alpha-alerte',
            color: '#00FF00',
            message: "SUCCESSFUL: Job '${env.JOB_NAME} [${env.BUILD_NUMBER}]' (${env.BUILD_URL})"
        )
    }
    failure {
        slackSend(
            channel: '#jenkins-eazytraining-alpha-alerte',
            color: '#FF0000',
            message: "FAILED: Job '${env.JOB_NAME} [${env.BUILD_NUMBER}]' (${env.BUILD_URL})"
        )
    }
}
