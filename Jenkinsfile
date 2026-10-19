pipeline {
    agent any

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
                sh 'docker compose build'
            }
        }

        stage('CD - Deploy') {
            steps {
                sh 'docker compose down || true'
                sh 'docker compose up -d'
            }
        }

        stage('CD - Verify Containers') {
            steps {
                sh '''
                    sleep 15
                    docker compose ps
                '''
            }
        }

        stage('CD - Health Check') {
            steps {
                sh '''
                    for i in $(seq 1 12); do
                        curl -sf http://localhost:8081/entreprise/all > /dev/null && break
                        echo "Backend not ready yet (try $i/12)..."
                        sleep 5
                    done
                    curl -f http://localhost:8081/entreprise/all
                    curl -f http://localhost:4200
                '''
            }
        }
    }

    post {
        success {
            echo 'CI/CD terminé avec succès : application testée, analysée et déployée.'
        }
        failure {
            echo 'Le pipeline a échoué. Vérifiez les logs ci-dessus.'
        }
    }
}