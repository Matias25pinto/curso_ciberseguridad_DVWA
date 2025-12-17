pipeline {
    agent any

    environment {
        SONAR_HOST_URL = 'http://sonarqube:9000'
    }

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
                
                // Usando credenciales de Jenkins para el token
                withCredentials([string(credentialsId: 'sonarqube-token', variable: 'SONAR_TOKEN')]) {
                    sh '''
                        cd dvwa
                        sonar-scanner \
                            -Dsonar.projectKey=dvwa-app \
                            -Dsonar.projectName="DVWA Security App" \
                            -Dsonar.sources=. \
                            -Dsonar.host.url=${SONAR_HOST_URL} \
                            -Dsonar.login=${SONAR_TOKEN} \
                            -Dsonar.php.file.suffixes=.php,.php3,.php4,.php5,.phtml \
                            -Dsonar.exclusions=**/vendor/**
                    '''
                }
            }
        }

        stage('Quality Gate') {
            steps {
                timeout(time: 5, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: false
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
}
