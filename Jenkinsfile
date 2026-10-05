pipeline {

    agent any

    stages {

        stage('Clone') {
            steps {
                git 'https://github.com/Alaa-Rami/DevOps-AppGestionDesProjets.git'
            }
        }

        stage('Build Backend') {
            steps {
                dir('backend') {
                    sh './mvnw clean package -DskipTests'
                }
            }
        }

        stage('Build Docker Images') {
            steps {
                sh 'docker compose build'
            }
        }

        stage('Deploy') {
            steps {
                sh 'docker compose up -d'
            }
        }

    }

}
