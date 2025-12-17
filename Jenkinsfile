pipeline {
    agent any

    stages {
        stage('Checkout') {
            steps {
                sh 'rm -rf dvwa || true'
                sh 'git clone https://github.com/Matias25pinto/curso_ciberseguridad_DVWA dvwa'
                stash name: 'dvwa-code', includes: 'dvwa/**'
            }
        }

        stage('SonarQube Analysis') {
            agent {
                docker {
                    image 'sonarsource/sonar-scanner-cli:latest'
                    args '--network jenkins-network -u root'
                }
            }
            steps {
                unstash 'dvwa-code'
                
                // Usando el token de SonarQube
                withCredentials([string(credentialsId: 'sonarqube-token', variable: 'SONAR_TOKEN')]) {
                    sh '''
                        cd dvwa
                        sonar-scanner \
                          -Dsonar.projectKey=DVWA-Security-App \
                          -Dsonar.sources=. \
                          -Dsonar.host.url=http://sonarqube:9000 \
                          -Dsonar.token=${SONAR_TOKEN} \
                          -Dsonar.php.file.suffixes=.php \
                          -Dsonar.exclusions=**/vendor/**
                    '''
                }
            }
        }

        stage('Quality Gate Check') {
            steps {
                script {
                    timeout(time: 5, unit: 'MINUTES') {
                        def qg = waitForQualityGate()
                        if (qg.status != 'OK') {
                            echo "⚠️  Quality Gate no aprobado: ${qg.status}"
                            currentBuild.result = 'UNSTABLE'
                        }
                    }
                }
            }
        }

        stage('Build & Deploy') {
            steps {
                sh 'cd dvwa && docker build -t dvwa-app:latest .'
                sh 'docker rm -f dvwa-app || true'
                sh 'docker run -d --name dvwa-app -p 8082:80 dvwa-app:latest'
            }
        }
    }
    
    post {
        always {
            echo "✅ Pipeline completado"
            echo "📊 Dashboard de SonarQube: http://localhost:9000/dashboard?id=DVWA-Security-App"
            echo "🌐 DVWA desplegado en: http://localhost:8082"
        }
    }
}
