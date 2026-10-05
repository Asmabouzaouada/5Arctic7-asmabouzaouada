pipeline {
    agent any
 
    environment {
        // ====== CHANGEZ ======
        PROJECT_KEY    = 'Mon-projet'
        DOCKERHUB_USER = 'asmabouzaouada'
        // format imposé : asmabouzaouada_5arctic7_monprojet (minuscules pour Docker Hub)
        IMAGE_BASE     = 'asmabouzaouada-5arctic7-monprojet'
        // =====================
        BACKEND_IMAGE  = "${DOCKERHUB_USER}/${IMAGE_BASE}-backend"
        FRONTEND_IMAGE = "${DOCKERHUB_USER}/${IMAGE_BASE}-frontend"
        IMAGE_TAG      = "${BUILD_NUMBER}"
        SPRING_DATASOURCE_PASSWORD = credentials('mysql-root-password')
    }
 
    stages {
        stage('GIT') {
            steps { checkout scm }
        }
 
        stage('Build') {
            steps { dir('backend') { sh 'mvn clean compile' } }
        }
 
        stage('Tests') {
            steps { dir('backend') { sh 'mvn test' } }
            post {
                always {
                    junit allowEmptyResults: true, testResults: 'backend/target/surefire-reports/*.xml'
                }
            }
        }
 
        stage('SonarQube') {
            steps {
                dir('backend') {
                    withSonarQubeEnv('SonarQube') {
                        sh 'mvn org.sonarsource.scanner.maven:sonar-maven-plugin:sonar -Dsonar.projectKey=$PROJECT_KEY -Dsonar.token=$SONAR_AUTH_TOKEN'
                    }
                }
            }
        }
 
        // Optionnel : nécessite un webhook SonarQube -> http://JENKINS:8080/sonarqube-webhook/
        // stage('Quality Gate') {
        //     steps { timeout(time: 5, unit: 'MINUTES') { waitForQualityGate abortPipeline: true } }
        // }
 
        stage('Package') {
            steps { dir('backend') { sh 'mvn package -DskipTests' } }
        }
 
        stage('Docker Build') {
            steps {
                sh '''
                  docker build -t $BACKEND_IMAGE:$IMAGE_TAG -t $BACKEND_IMAGE:latest backend
                  docker build -t $FRONTEND_IMAGE:$IMAGE_TAG -t $FRONTEND_IMAGE:latest frontend
                '''
            }
        }
 
        stage('Docker Push') {
            steps {
                withCredentials([usernamePassword(credentialsId: 'dockerhub-credentials',
                                                  usernameVariable: 'DH_USER', passwordVariable: 'DH_PASS')]) {
                    sh '''
                      echo "$DH_PASS" | docker login -u "$DH_USER" --password-stdin
                      docker push $BACKEND_IMAGE:$IMAGE_TAG
                      docker push $BACKEND_IMAGE:latest
                      docker push $FRONTEND_IMAGE:$IMAGE_TAG
                      docker push $FRONTEND_IMAGE:latest
                      docker logout
                    '''
                }
            }
        }
 
        stage('Deploy Kubernetes') {
            steps {
                withCredentials([file(credentialsId: 'kubeconfig', variable: 'KUBECONFIG')]) {
                    sh '''
                      kubectl apply -f k8s/namespace.yaml
 
                      kubectl -n devops create secret generic mysql-secret \
                        --from-literal=MYSQL_ROOT_PASSWORD="$SPRING_DATASOURCE_PASSWORD" \
                        --dry-run=client -o yaml | kubectl apply -f -
 
                      kubectl apply -f k8s/mysql.yaml
                      kubectl -n devops rollout status deploy/mysql --timeout=180s
 
                      sed "s|__IMAGE__|$BACKEND_IMAGE:$IMAGE_TAG|g"  k8s/backend.yaml  | kubectl apply -f -
                      sed "s|__IMAGE__|$FRONTEND_IMAGE:$IMAGE_TAG|g" k8s/frontend.yaml | kubectl apply -f -
 
                      kubectl -n devops rollout status deploy/backend  --timeout=300s
                      kubectl -n devops rollout status deploy/frontend --timeout=180s
                      kubectl -n devops get pods,svc
                    '''
                }
            }
        }
    }
 
    post {
        success { echo "Pipeline OK - images ${BACKEND_IMAGE}:${IMAGE_TAG} / ${FRONTEND_IMAGE}:${IMAGE_TAG} déployées" }
        failure { echo 'Pipeline en échec - voir les logs' }
        // always { emailext subject: "Build ${currentBuild.currentResult}: ${env.JOB_NAME} #${env.BUILD_NUMBER}", body: "${env.BUILD_URL}", to: 'vous@mail.com' }
    }
}
 
