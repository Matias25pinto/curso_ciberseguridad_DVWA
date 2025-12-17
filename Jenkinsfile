pipeline {
    agent any

    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('SonarQube Analysis with JSON Report') {
            agent {
                docker {
                    image 'sonarsource/sonar-scanner-cli:latest'
                    args '--network jenkins-network -u root'
                }
            }
            steps {
                script {
                    // Función para obtener reporte JSON de SonarQube
                    def getSonarQubeReport = { token, projectKey ->
                        sh """
                            # Análisis principal
                            sonar-scanner \
                              -Dsonar.projectKey=${projectKey} \
                              -Dsonar.sources=. \
                              -Dsonar.host.url=http://sonarqube:9000 \
                              -Dsonar.token=${token}
                            
                            # Esperar procesamiento
                            sleep 15
                            
                            # Obtener issues en formato JSON
                            curl -s -u ${token}: \
                              "http://sonarqube:9000/api/issues/search?componentKeys=${projectKey}&resolved=false&ps=1000" \
                              -o sonarqube-issues.json
                            
                            # Obtener medidas
                            curl -s -u ${token}: \
                              "http://sonarqube:9000/api/measures/component?component=${projectKey}&metricKeys=bugs,vulnerabilities,code_smells,security_hotspots,coverage" \
                              -o sonarqube-measures.json
                            
                            # Crear reporte resumido
                            echo '{
                              "project": "${projectKey}",
                              "timestamp": "'$(date -Iseconds)'",
                              "dashboard_url": "http://sonarqube:9000/dashboard?id=${projectKey}"
                            }' > sonarqube-summary.json
                            
                            # Parsear y agregar datos
                            jq -s '.[0] * {issues: .[1], measures: .[2]}' \
                              sonarqube-summary.json \
                              sonarqube-issues.json \
                              sonarqube-measures.json > sonarqube-report.json
                        """
                    }
                    
                    // Usar la función
                    withCredentials([string(credentialsId: 'sonarqube-token', variable: 'SONAR_TOKEN')]) {
                        getSonarQubeReport(SONAR_TOKEN, 'DVWA-Security-App')
                    }
                    
                    // Contar y mostrar hallazgos
                    sh '''
                        echo "=== Estadísticas de SonarQube ==="
                        if [ -f sonarqube-report.json ]; then
                            BUGS=$(jq '.measures.component.measures[] | select(.metric=="bugs") | .value' sonarqube-report.json)
                            VULNS=$(jq '.measures.component.measures[] | select(.metric=="vulnerabilities") | .value' sonarqube-report.json)
                            SMELLS=$(jq '.measures.component.measures[] | select(.metric=="code_smells") | .value' sonarqube-report.json)
                            HOTSPOTS=$(jq '.measures.component.measures[] | select(.metric=="security_hotspots") | .value' sonarqube-report.json)
                            
                            echo "Bugs: ${BUGS:-0}"
                            echo "Vulnerabilidades: ${VULNS:-0}"
                            echo "Code Smells: ${SMELLS:-0}"
                            echo "Security Hotspots: ${HOTSPOTS:-0}"
                        fi
                    '''
                }
                
                // Archivar todos los JSONs
                archiveArtifacts artifacts: 'sonarqube-*.json', fingerprint: true
            }
        }
        
        stage('Build and Deploy') {
            steps {
                sh 'docker build -t dvwa-app:latest .'
                sh 'docker rm -f dvwa-app || true'
                sh 'docker run -d --name dvwa-app -p 8082:80 dvwa-app:latest'
            }
        }
    }
}
