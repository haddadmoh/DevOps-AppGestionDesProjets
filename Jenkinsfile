pipeline {
    agent any
    stages {
        stage('GIT') {
            steps {
                git branch: 'main', url: 'https://github.com/haddadmoh/DevOps-AppGestionDesProjets.git'
            }
        }
        stage('Build') {
            steps {
                dir('backend') {
                    sh 'mvn clean compile'
                }
            }
        }
        stage('Tests') {
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