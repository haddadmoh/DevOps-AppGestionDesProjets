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
                    sh 'mvn test'
                }
            }
        }
        stage('SonarQube') {
            steps {
                dir('backend') {
                    withSonarQubeEnv('SonarQube') {
                        sh 'mvn sonar:sonar'
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