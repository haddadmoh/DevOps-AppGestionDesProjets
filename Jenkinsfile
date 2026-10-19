pipeline {
    agent any

    environment {
        TAG       = "${BUILD_NUMBER}"
        // Gives DOCKERHUB_USR and DOCKERHUB_PSW automatically
        DOCKERHUB = credentials('dockerhub-credentials')
    }

    stages {

        // =====================================================
        //                         CI
        // =====================================================
        stage('CI') {
            stages {

                stage('GIT') {
                    steps {
                        checkout scm
                    }
                }

                stage('Build') {
                    steps {
                        dir('backend') {
                            sh 'mvn clean compile'
                        }
                    }
                }

                stage('Tests (JUnit + JaCoCo)') {
                    steps {
                        dir('backend') {
                            withEnv(['DB_HOST=localhost', 'DB_PORT=3307', 'DB_NAME=ci_db', 'DB_USER=root', 'DB_PASSWORD=root']) {
                                sh 'mvn test'
                            }
                        }
                    }
                }

                stage('SonarQube') {
                    steps {
                        dir('backend') {
                            withSonarQubeEnv('SonarQube') {
                                sh 'mvn org.sonarsource.scanner.maven:sonar-maven-plugin:5.1.0.4751:sonar'
                            }
                        }
                    }
                }

                stage('Quality Gate') {
                    steps {
                        timeout(time: 5, unit: 'MINUTES') {
                            waitForQualityGate abortPipeline: true
                        }
                    }
                }

                stage('Package') {
                    steps {
                        dir('backend') {
                            sh 'mvn package -DskipTests'
                        }
                    }
                }
            }
        }

        // =====================================================
        //                         CD
        // =====================================================
        stage('CD') {
            stages {

                stage('Build Images') {
                    steps {
                        sh 'docker build -t $DOCKERHUB_USR/backend:$TAG ./backend'
                        sh 'docker build -t $DOCKERHUB_USR/frontend:$TAG ./frontend'
                    }
                }

                stage('Push DockerHub') {
                    steps {
                        sh 'echo $DOCKERHUB_PSW | docker login -u $DOCKERHUB_USR --password-stdin'
                        sh 'docker push $DOCKERHUB_USR/backend:$TAG'
                        sh 'docker push $DOCKERHUB_USR/frontend:$TAG'
                    }
                }

                stage('Deploy K8s') {
                    steps {
                        // namespace (no error if it already exists)
                        sh 'kubectl create namespace devops --dry-run=client -o yaml | kubectl apply -f -'

                        // 1. MySQL
                        sh 'kubectl apply -f k8s/mysql-deployment.yaml'
                        sh 'kubectl rollout status deployment/mysql -n devops --timeout=3m'

                        // 2. Backend (image line replaced with this build's tag)
                        sh 'sed "s|image: .*/backend:.*|image: $DOCKERHUB_USR/backend:$TAG|" k8s/spring-deployment.yaml | kubectl apply -f -'
                        sh 'kubectl rollout status deployment/backend -n devops --timeout=5m'

                        // 3. Frontend (needs the "backend" service to exist, so it comes last)
                        sh 'sed "s|image: .*/frontend:.*|image: $DOCKERHUB_USR/frontend:$TAG|" k8s/frontend-deployment.yaml | kubectl apply -f -'
                        sh 'kubectl rollout status deployment/frontend -n devops --timeout=5m'
                    }
                }

                stage('Verify') {
                    steps {
                        sh '''
                            kubectl get pods -n devops -o wide
                            kubectl get svc -n devops

                            NODE_IP=$(kubectl get nodes -o jsonpath='{.items[0].status.addresses[?(@.type=="InternalIP")].address}')
                            for i in $(seq 1 12); do
                                curl -sf http://$NODE_IP:30081/entreprise/all > /dev/null && break
                                echo "Backend not ready yet (try $i/12)..."
                                sleep 5
                            done
                            curl -f http://$NODE_IP:30081/entreprise/all
                            curl -fI http://$NODE_IP:30080
                        '''
                    }
                }
            }
        }
    }

    post {
        success {
            echo 'CI/CD terminé : images poussées sur DockerHub et application déployée sur Kubernetes.'
        }
        failure {
            echo 'Le pipeline a échoué. Vérifiez les logs ci-dessus.'
        }
        always {
            sh 'docker logout || true'
        }
    }
}