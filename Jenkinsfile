pipeline {
    agent any
    environment {
        TAG = sh(script: 'git describe --abbrev=0',,returnStdout: true).trim()
    }
    stages {
        stage('build da imagem docker') { 
            steps {
                sh 'docker build -t devops/app:${TAG} .'
            }
        }
        stage('subir docker compose - redis e app') {
            steps {
                sh 'docker compose up --build -d'
            }
        }
        stage('sleep para subida de containers') {
            steps {
                sh 'sleep 10'
            }
        }
        stage('Sonarqube validation') {
            steps {
                script {
                    scannerHome = tool 'sonar-scanner';
                }
                withSonarQubeEnv('sonar-server') {
                    sh "${scannerHome}/bin/sonar-scanner -Dsonar.projectKey=redis-app -Dsonar.sources=. -Dsonar.host.url=${env.SONAR_HOST_URL} -Dsonar.login=${env.SONAR_AUTH_TOKEN}"
                }
                sh 'sleep 10'
            }
        }
        stage('Quality Gate') {
            steps {
                waitForQualityGate abortPipeline: true
            }
        }
        stage('teste da app') {
            steps {
                sh 'chmod +x teste-app.sh'
                sh './teste-app.sh'
            }
        }
        stage('shutdown containers') {
            steps {
                sh 'docker compose down'
            }
        }
        stage('upload docker images to nexus') {
            steps {
                script {
                    withCredentials([usernamePassword(credentialsId: 'nexus-user', usernameVariable: 'USERNAME', passwordVariable: 'PASSWORD')]) {
                        sh 'docker login -u $USERNAME -p $PASSWORD ${NEXUS_URL}'
                        sh 'docker tag devops/app:${TAG} ${NEXUS_URL}/devops/app:${TAG}'
                        sh 'docker push ${NEXUS_URL}/devops/app:${TAG}'
                    }
                }
            }
        }
        stage('Apply k8s files') {
            steps {
                sh "sed -i -e 's#TAG#${TAG}#' ./k3s/redis-app.yaml;"
                sh '/usr/local/bin/kubectl apply -f ./k3s/redis.yaml'
                sh '/usr/local/bin/kubectl apply -f ./k3s/redis-app.yaml'
            }
        }
    }

}
