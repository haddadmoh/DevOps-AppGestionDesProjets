pipeline {
    agent any

    stages {

        stage('Checkout') {
            steps {
                git branch: 'main',
                    url: 'https://github.com/haddadmoh/DevOps-AppGestionDesProjets.git'
            }
        }

        stage('Build Backend Image') {
            steps {
                dir('backend') {
                    sh 'docker build -t gestion-projets-backend:latest .'
                }
            }
        }

        stage('Build Frontend Image') {
            steps {
                dir('frontend') {
                    sh 'docker build -t gestion-projets-frontend:latest .'
                }
            }
        }

        stage('Deploy with Docker Compose') {
            steps {
                sh 'docker compose down || true'
                sh 'docker compose up -d --build'
            }
        }

        stage('Verify Containers') {
            steps {
                sh '''
                    sleep 15
                    docker compose ps
                '''
            }
        }

        stage('Health Check') {
            steps {
                sh '''
                    curl -f http://localhost:8080/entreprise/all || echo "Backend not responding"
                    curl -f http://localhost:4200 || echo "Frontend not responding"
                '''
            }
        }
    }

    post {
        success {
            echo 'Pipeline terminé avec succès : application déployée.'
        }
        failure {
            echo 'Le pipeline a échoué. Vérifiez les logs ci-dessus.'
        }
    }
}