// ============================================================
// Jenkinsfile - Pipeline CI/CD pour PayMyBuddy
// Description : Build, test, analyse qualité et déploiement
//               automatisé sur les environnements staging et prod
//               via Docker et SSH sur instances AWS EC2.
// Modèle de branche : Gitflow
//   - main        : pipeline complet (CI + CD)
//   - autres      : pipeline partiel (CI uniquement, sans déploiement)
// ============================================================

// ------------------------------------------------------------
// Fonction réutilisable pour le déploiement sur un serveur
// Paramètres :
//   - hostname      : IP ou nom DNS du serveur cible
//   - sshCredential : ID du credential SSH Jenkins
// ------------------------------------------------------------
def deployToServer(String hostname, String sshCredential) {
    // Installation des outils SSH et curl dans le conteneur Alpine
    sh 'apk --no-cache add openssh-client curl'

    // Utilisation du credential SSH pour s'authentifier sur le serveur distant
    sshagent(credentials: [sshCredential]) {
        sh """
            set -e

            # Création du répertoire SSH et ajout du serveur aux hôtes connus
            mkdir -p ~/.ssh
            chmod 700 ~/.ssh
            ssh-keyscan -T 30 -t rsa ${hostname} >> ~/.ssh/known_hosts

            # Copie du script SQL d'initialisation de la base de données sur le serveur
            scp src/main/resources/database/create.sql ubuntu@${hostname}:/home/ubuntu/create.sql

            # Connexion SSH au serveur et exécution des commandes de déploiement
            ssh ubuntu@${hostname} "
                set -e

                # Installation de Docker si non présent sur le serveur
                if ! command -v docker >/dev/null 2>&1; then
                    curl -fsSL https://get.docker.com -o install-docker.sh
                    sudo sh install-docker.sh
                    sudo usermod -aG docker ubuntu
                fi

                # Authentification sur DockerHub
                echo \$DOCKERHUB_AUTH_PSW | docker login -u \$DOCKERHUB_AUTH_USR --password-stdin

                # Récupération de la dernière image depuis DockerHub
                docker pull \$ID_DOCKER/\$IMAGE_NAME:\$IMAGE_TAG

                # Suppression des anciens conteneurs et du réseau pour repartir proprement
                docker rm -f paymybuddy mysql-db || true
                docker network rm app-network || true

                # Création du volume persistant pour les données MySQL et du réseau applicatif
                docker volume create mysql-data || true
                docker network create app-network

                # Démarrage du conteneur MySQL avec le script d'init de la base de données
                docker run -d --name mysql-db \\
                    --network app-network \\
                    -e MYSQL_ROOT_PASSWORD=\$MYSQL_ROOT_PASSWORD \\
                    -v mysql-data:/var/lib/mysql \\
                    -v /home/ubuntu/create.sql:/docker-entrypoint-initdb.d/create.sql \\
                    mysql:8.0

                # Attente que MySQL soit prêt avant de démarrer l'application
                echo 'Waiting for MySQL...'
                until docker exec mysql-db mysqladmin ping -h localhost --silent; do
                    sleep 5
                done

                # Démarrage du conteneur de l'application PayMyBuddy
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
    // Pas d'agent global : chaque stage définit son propre agent Docker
    agent none

    // ------------------------------------------------------------
    // Variables d'environnement globales
    // Les credentials sont injectés de manière sécurisée via Jenkins
    // Les IPs des serveurs sont définies ici (à externaliser dans un .env)
    // ------------------------------------------------------------
    environment {
        DOCKERHUB_AUTH          = credentials('DOCKERHUB_AUTH')       // Login/mot de passe DockerHub
        ID_DOCKER               = "${DOCKERHUB_AUTH_USR}"             // Nom d'utilisateur DockerHub
        IMAGE_NAME              = "paymybuddy"                        // Nom de l'image Docker
        IMAGE_TAG               = "latest"                            // Tag de l'image Docker
        HOSTNAME_DEPLOY_STAGING = "100.48.221.157"                    // IP du serveur staging (AWS EC2)
        HOSTNAME_DEPLOY_PROD    = "44.222.235.73"                     // IP du serveur prod (AWS EC2)
        SONAR_TOKEN             = credentials('jenkins-sonar')        // Token SonarCloud
        MYSQL_ROOT_PASSWORD     = credentials('MYSQL_ROOT_PASSWORD')  // Mot de passe root MySQL
    }

    stages {

        // ------------------------------------------------------------
        // Stage 1 : Nettoyage de l'espace de travail
        // Supprime les fichiers du build précédent pour éviter
        // les conflits de permissions liés aux conteneurs Docker root
        // ------------------------------------------------------------
        stage('Clean Workspace') {
            agent any
            steps {
                cleanWs()
            }
        }

        // ------------------------------------------------------------
        // Stage 2 : Exécution des tests unitaires
        // Utilise Maven dans un conteneur pour exécuter les tests
        // ------------------------------------------------------------
        stage('Tests') {
            agent {
                docker {
                    image 'maven:3.8.6-amazoncorretto-17'
                    args '-v /root/.m2:/root/.m2' // Cache Maven monté pour accélérer les builds
                }
            }
            steps {
                sh 'mvn test'
            }
        }

        // ------------------------------------------------------------
        // Stage 3 : Analyse de la qualité du code avec SonarCloud
        // Envoie les métriques de code (bugs, vulnerabilités, coverage)
        // vers SonarCloud pour analyse statique
        // ------------------------------------------------------------
        stage('Code Quality') {
            agent {
                docker {
                    image 'maven:3.8.6-amazoncorretto-17'
                    args '-v /root/.m2:/root/.m2'
                }
            }
            steps {
                // Le wrapper withSonarQubeEnv est obligatoire pour que
                // waitForQualityGate puisse retrouver l'analyse
                withSonarQubeEnv('sonarcloud') {
                    sh """
                        mvn sonar:sonar \
                          -Dsonar.projectKey=paymybuddy \
                          -Dsonar.organization=alpha-jenkins \
                          -Dsonar.host.url=https://sonarcloud.io \
                          -Dsonar.login=${SONAR_TOKEN}
                    """
                }
            }
        }

        // ------------------------------------------------------------
        // Stage 4 : Vérification du Quality Gate SonarCloud
        // Attend la réponse de SonarCloud via webhook et bloque
        // le pipeline si le Quality Gate n'est pas passé
        // ------------------------------------------------------------
        stage('Quality Gate') {
            agent any
            steps {
                timeout(time: 2, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: true
                }
            }
        }

        // ------------------------------------------------------------
        // Stage 5 : Build de l'artifact Maven
        // Compile et package l'application en .jar
        // Les tests sont ignorés (déjà exécutés au stage 2)
        // ------------------------------------------------------------
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

        // ------------------------------------------------------------
        // Stage 6 : Build et push de l'image Docker sur DockerHub
        // Monte le socket Docker du host pour accéder au daemon Docker
        // ------------------------------------------------------------
        stage('Build & Push Docker Image') {
            agent {
                docker {
                    image 'docker:latest'
                    args '-u root -v /var/run/docker.sock:/var/run/docker.sock'
                }
            }
            steps {
                sh """
                    # Construction de l'image Docker à partir du Dockerfile
                    docker build -t ${ID_DOCKER}/${IMAGE_NAME}:${IMAGE_TAG} .

                    # Authentification et push vers DockerHub
                    echo \$DOCKERHUB_AUTH_PSW | docker login -u \$DOCKERHUB_AUTH_USR --password-stdin
                    docker push ${ID_DOCKER}/${IMAGE_NAME}:${IMAGE_TAG}
                """
            }
        }

        // ------------------------------------------------------------
        // Stage 7 : Déploiement sur le serveur Staging
        // Exécuté uniquement sur la branche main (Gitflow)
        // Utilise la fonction deployToServer définie en haut du fichier
        // ------------------------------------------------------------
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

        // ------------------------------------------------------------
        // Stage 8 : Test de validation du serveur Staging
        // Vérifie que l'application répond bien sur le port 80
        // Un code HTTP 200 ou 302 (redirect login) est accepté
        // ------------------------------------------------------------
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
                    sleep 15  # Attente du démarrage complet de l'application
                    STATUS=\$(curl -s -o /dev/null -w "%{http_code}" http://${HOSTNAME_DEPLOY_STAGING})
                    echo "HTTP Status: \$STATUS"
                    [ "\$STATUS" = "200" ] || [ "\$STATUS" = "302" ]
                """
            }
        }

        // ------------------------------------------------------------
        // Stage 9 : Déploiement sur le serveur de Production
        // Exécuté uniquement sur la branche main, après validation staging
        // ------------------------------------------------------------
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

        // ------------------------------------------------------------
        // Stage 10 : Test de validation du serveur de Production
        // Même vérification HTTP que pour le staging
        // ------------------------------------------------------------
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
                    sleep 15  # Attente du démarrage complet de l'application
                    STATUS=\$(curl -s -o /dev/null -w "%{http_code}" http://${HOSTNAME_DEPLOY_PROD})
                    echo "HTTP Status: \$STATUS"
                    [ "\$STATUS" = "200" ] || [ "\$STATUS" = "302" ]
                """
            }
        }
    }

    // ------------------------------------------------------------
    // Post-actions : Notifications Slack selon le résultat du pipeline
    // - SUCCESS  : vert  (#00FF00)
    // - FAILURE  : rouge (#FF0000)
    // - UNSTABLE : orange (#FFA500) — ex: Quality Gate warning
    // ------------------------------------------------------------
    post {
        success {
            slackSend(
                channel: '#jenkins-eazytraining-alpha-alerte',
                color: '#00FF00',
                message: "✅ SUCCESS: ${env.JOB_NAME} [${env.BUILD_NUMBER}] ${env.BUILD_URL}"
            )
        }
        failure {
            slackSend(
                channel: '#jenkins-eazytraining-alpha-alerte',
                color: '#FF0000',
                message: "❌ FAILED: ${env.JOB_NAME} [${env.BUILD_NUMBER}] ${env.BUILD_URL}"
            )
        }
        unstable {
            slackSend(
                channel: '#jenkins-eazytraining-alpha-alerte',
                color: '#FFA500',
                message: "⚠️ UNSTABLE: ${env.JOB_NAME} [${env.BUILD_NUMBER}] ${env.BUILD_URL}"
            )
        }
    }
}
