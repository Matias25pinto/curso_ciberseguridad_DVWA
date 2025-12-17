pipeline {
    agent any

    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('SonarQube Quick Test') {
            agent {
                docker {
                    image 'sonarsource/sonar-scanner-cli:latest'
                    args '--network jenkins-network'
                }
            }
            steps {
                withCredentials([string(credentialsId: 'sonarqube-token', variable: 'SONAR_TOKEN')]) {
                    sh '''
                        # Análisis simple
                        sonar-scanner \
                          -Dsonar.projectKey=DVWA-Security-App \
                          -Dsonar.sources=. \
                          -Dsonar.host.url=http://sonarqube:9000 \
                          -Dsonar.token=${SONAR_TOKEN}
                        
                        # Obtener solo 100 issues para prueba
                        curl -s -u ${SONAR_TOKEN}: \
                          "http://sonarqube:9000/api/issues/search?componentKeys=DVWA-Security-App&resolved=false&ps=100" \
                          -o sonarqube-test.json
                        
                        # Verificar si hay errores
                        if jq -e '.errors' sonarqube-test.json >/dev/null 2>&1; then
                            echo "Error en la API:"
                            jq '.errors' sonarqube-test.json
                            exit 1
                        fi
                        
                        # Mostrar resumen
                        echo "=== Resumen de SonarQube ==="
                        TOTAL=$(jq '.total' sonarqube-test.json 2>/dev/null || echo "0")
                        echo "Total issues encontrados: $TOTAL"
                    '''
                }
            }
        }
    }
}
