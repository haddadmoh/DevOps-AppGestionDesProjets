pipeline {
    agent any

    environment {
        DOCKERHUB_USER = 'mouhamedhaddad'
        TAG            = "${BUILD_NUMBER}"
        K8S_DIR        = '/home/vagrant/homework_lab4/k8s'
    }

    stages {

        // =====================================================
        //                         CI
        // =====================================================

        stage('CI - Checkout') {
            steps {
                git branch: 'main',
                    url: 'https://github.com/haddadmoh/DevOps-AppGestionDesProjets.git'
            }
        }

        stage('CI - Build') {
            steps {
                dir('backend') {
                    sh 'mvn clean compile'
                }
            }
        }

        stage('CI - Tests (JUnit + JaCoCo)') {
            steps {
                dir('backend') {
                    withEnv(['DB_HOST=localhost', 'DB_PORT=3307', 'DB_NAME=ci_db', 'DB_USER=root', 'DB_PASSWORD=root']) {
                        sh 'mvn test'
                    }
                }
            }
        }

        stage('CI - SonarQube Analysis') {
            steps {
                dir('backend') {
                    withSonarQubeEnv('SonarQube') {
                        sh 'mvn org.sonarsource.scanner.maven:sonar-maven-plugin:5.1.0.4751:sonar'
                    }
                }
            }
        }

        stage('CI - Quality Gate') {
            steps {
                timeout(time: 5, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: true
                }
            }
        }

        stage('CI - Package') {
            steps {
                dir('backend') {
                    sh 'mvn package -DskipTests'
                }
            }
        }

        // =====================================================
        //                         CD
        // =====================================================

        stage('CD - Build Docker Images') {
            steps {
                sh 'docker build -t $DOCKERHUB_USER/backend:$TAG ./backend'
                sh 'docker build -t $DOCKERHUB_USER/frontend:$TAG ./frontend'
            }
        }

        stage('CD - Push DockerHub') {
            steps {
                withCredentials([usernamePassword(
                        credentialsId: 'dockerhub-credentials',
                        usernameVariable: 'HUB_USER',
                        passwordVariable: 'HUB_PASS')]) {
                    sh 'echo $HUB_PASS | docker login -u $HUB_USER --password-stdin'
                    sh 'docker push $DOCKERHUB_USER/backend:$TAG'
                    sh 'docker push $DOCKERHUB_USER/frontend:$TAG'
                }
            }
        }

        stage('CD - Deploy K8s') {
            steps {
                sh '''
                    kubectl apply -f $K8S_DIR/mysql-deployment.yaml

                    sed -E "s|image: .*/backend:.*|image: $DOCKERHUB_USER/backend:$TAG|" $K8S_DIR/spring-deployment.yaml | kubectl apply -f -
                    sed -E "s|image: .*/frontend:.*|image: $DOCKERHUB_USER/frontend:$TAG|" $K8S_DIR/frontend-deployment.yaml | kubectl apply -f -
                '''
            }
        }

        stage('CD - Verify Pods') {
            steps {
                sh '''
                    sleep 15
                    kubectl get pods -n devops
                    kubectl get svc -n devops
                '''
            }
        }

        stage('CD - Health Check') {
            steps {
                sh '''
                    NODE_IP=$(minikube ip)
                    for i in $(seq 1 24); do
                        curl -sf http://$NODE_IP:30081/entreprise/all > /dev/null && break
                        echo "Backend not ready yet (try $i/24)..."
                        sleep 5
                    done
                    curl -f http://$NODE_IP:30081/entreprise/all
                    curl -f http://$NODE_IP:30080
                '''
            }
        }
    }

    post {
        success {
            echo 'CI/CD terminé avec succès : application testée, analysée et déployée sur Kubernetes.'
        }
        failure {
            echo 'Le pipeline a échoué. Vérifiez les logs ci-dessus.'
        }
        always {
            sh 'docker logout || true'
        }
    }
}