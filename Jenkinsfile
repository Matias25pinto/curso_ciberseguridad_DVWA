pipeline {
    agent any

    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('SAST-SonarQube with Pagination') {
            agent {
                docker {
                    image 'sonarsource/sonar-scanner-cli:latest'
                    args '--network jenkins-network -u root'
                }
            }
            steps {
                script {
                    withCredentials([string(credentialsId: 'sonarqube-token', variable: 'SONAR_TOKEN')]) {
                        sh '''
                            # Instalar jq para procesar JSON
                            apt-get update && apt-get install -y jq curl
                            
                            # Ejecutar análisis
                            sonar-scanner \
                              -Dsonar.projectKey=DVWA-Security-App \
                              -Dsonar.sources=. \
                              -Dsonar.host.url=http://sonarqube:9000 \
                              -Dsonar.token=${SONAR_TOKEN}
                            
                            # Esperar procesamiento
                            sleep 15
                            
                            # Obtener el total de issues primero
                            echo "Obteniendo total de issues..."
                            TOTAL_ISSUES=$(curl -s -u ${SONAR_TOKEN}: \
                              "http://sonarqube:9000/api/issues/search?componentKeys=DVWA-Security-App&resolved=false&ps=1" \
                              | jq '.total' || echo "0")
                            
                            echo "Total de issues: $TOTAL_ISSUES"
                            
                            # Si hay más de 500 issues, necesitamos paginación
                            PAGE_SIZE=500
                            PAGES=$(( ($TOTAL_ISSUES + $PAGE_SIZE - 1) / $PAGE_SIZE ))
                            
                            echo "Recuperando $PAGES páginas de issues..."
                            
                            # Crear archivo base
                            echo '{"issues": [], "total": '$TOTAL_ISSUES'}' > all-issues.json
                            
                            # Recorrer todas las páginas
                            for (( PAGE=1; PAGE<=PAGES; PAGE++ )); do
                                echo "Obteniendo página $PAGE de $PAGES..."
                                
                                curl -s -u ${SONAR_TOKEN}: \
                                  "http://sonarqube:9000/api/issues/search?componentKeys=DVWA-Security-App&resolved=false&ps=${PAGE_SIZE}&p=${PAGE}" \
                                  -o "page-${PAGE}.json"
                                
                                # Combinar los issues
                                jq -s '.[0].issues += .[1].issues | .[0]' \
                                  all-issues.json page-${PAGE}.json > temp.json
                                mv temp.json all-issues.json
                            done
                            
                            # Obtener medidas también
                            curl -s -u ${SONAR_TOKEN}: \
                              "http://sonarqube:9000/api/measures/component?component=DVWA-Security-App&metricKeys=bugs,vulnerabilities,code_smells,security_hotspots" \
                              -o measures.json
                            
                            # Crear reporte final combinado
                            jq -s '.[0] * {measures: .[1]}' all-issues.json measures.json > sonarqube-full-report.json
                            
                            # Crear un resumen simple
                            echo "=== RESUMEN DE SONARQUBE ==="
                            echo "Total issues: $TOTAL_ISSUES"
                            
                            # Contar por severidad si hay issues
                            if [ "$TOTAL_ISSUES" -gt 0 ]; then
                                BLOCKER=$(jq '[.issues[] | select(.severity=="BLOCKER")] | length' all-issues.json)
                                CRITICAL=$(jq '[.issues[] | select(.severity=="CRITICAL")] | length' all-issues.json)
                                MAJOR=$(jq '[.issues[] | select(.severity=="MAJOR")] | length' all-issues.json)
                                MINOR=$(jq '[.issues[] | select(.severity=="MINOR")] | length' all-issues.json)
                                
                                echo "Blocker: $BLOCKER"
                                echo "Critical: $CRITICAL"
                                echo "Major: $MAJOR"
                                echo "Minor: $MINOR"
                            fi
                            
                            # Mostrar medidas
                            if [ -f measures.json ]; then
                                echo "=== MÉTRICAS ==="
                                jq -r '.component.measures[] | "\(.metric): \(.value)"' measures.json 2>/dev/null || true
                            fi
                            
                            # Para simplificar, también crear un archivo pequeño con solo el resumen
                            echo '{
                              "project": "DVWA-Security-App",
                              "total_issues": '$TOTAL_ISSUES',
                              "timestamp": "'$(date -Iseconds)'"
                            }' > sonarqube-summary.json
                        '''
                    }
                }
                
                // Archivar múltiples archivos JSON
                archiveArtifacts artifacts: 'sonarqube-*.json,all-issues.json,measures.json', fingerprint: true, allowEmptyArchive: true
            }
        }

        stage('Build & Deploy') {
            steps {
                sh 'docker build -t dvwa-app:latest .'
                sh 'docker rm -f dvwa-app || true'
                sh 'docker run -d --name dvwa-app -p 8082:80 dvwa-app:latest'
            }
        }
    }
}
