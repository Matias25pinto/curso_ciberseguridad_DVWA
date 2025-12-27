pipeline {
    agent any

    stages {
        stage('Checkout') {
            steps {
                echo "Obteniendo el código desde GitHub..."
                sh 'rm -rf dvwa || true'
                sh 'git clone https://github.com/Matias25pinto/curso_ciberseguridad_DVWA dvwa'
                stash name: 'dvwa-code', includes: 'dvwa/**'
            }
        }

        stage('SAST-SonarQube') {
            agent {
                docker {
                    image 'sonarsource/sonar-scanner-cli:latest'
                    args '--network jenkins-network -u root'
                }
            }
            steps {
                script {
                    unstash 'dvwa-code'

                    sh 'rm -f sonarqube.json || true'

                    withCredentials([string(credentialsId: 'sonarqube-token', variable: 'SONAR_TOKEN')]) {
                        sh '''
                            cd dvwa

                            # 1. Ejecutar análisis
                            sonar-scanner \
                            -Dsonar.projectKey=DVWA-Security-App \
                            -Dsonar.sources=. \
                            -Dsonar.host.url=http://sonarqube:9000 \
                            -Dsonar.token=${SONAR_TOKEN}

                            # 2. Esperar procesamiento
                            sleep 10

                            # 3. Generar reporte JSON
                            curl -s -u ${SONAR_TOKEN}: \
                            "http://sonarqube:9000/api/issues/search?componentKeys=DVWA-Security-App&resolved=false&ps=500" \
                            -o ../sonarqube.json
                        '''
                    }

                    sh '''
                        test -f sonarqube.json || echo "{}" > sonarqube.json
                        echo "=== Contenido sonarqube.json ==="
                        jq '.total' sonarqube.json || cat sonarqube.json
                    '''

                    archiveArtifacts artifacts: 'sonarqube.json', fingerprint: true
                }
            }
        }

        stage('Quality Gate (API)') {
            steps {
                script {
                    withCredentials([string(credentialsId: 'sonarqube-token', variable: 'SONAR_TOKEN')]) {
                        sh '''
                            echo "⏳ Esperando resultado del Quality Gate..."
                            sleep 10

                            RESPONSE=$(curl -s -u ${SONAR_TOKEN}: \
                            "http://sonarqube:9000/api/qualitygates/project_status?projectKey=DVWA-Security-App")

                            echo "Respuesta SonarQube:"
                            echo "$RESPONSE"

                            STATUS=$(echo "$RESPONSE" | grep -o '"status":"[^"]*"' | head -1 | sed 's/"status":"//;s/"//')

                            echo "Quality Gate status: $STATUS"

                            if [ "$STATUS" != "OK" ]; then
                                echo "❌ Quality Gate fallido"
                                exit 1
                            fi
                        '''
                    }
                }
            }
        }

        stage('Fail on Critical Issues') {
            steps {
                script {
                    sh '''
                        echo "🔍 Buscando issues CRITICAL o BLOCKER..."

                        if grep -q '"severity":"CRITICAL"' sonarqube.json || grep -q '"severity":"BLOCKER"' sonarqube.json; then
                            echo "❌ Se encontraron vulnerabilidades CRITICAL/BLOCKER"
                            exit 1
                        else
                            echo "✅ No se encontraron issues críticos"
                        fi
                    '''
                }
            }
        }

        stage('Build & Deploy') {
            steps {
                echo "Construyendo Docker image para DVWA..."
                sh 'cd dvwa && docker build -t dvwa-app:latest .'
                
                echo "Desplegando DVWA con Docker..."
                sh """
                    docker rm -f dvwa-app || true
                    docker run -d --name dvwa-app -p 8082:80 dvwa-app:latest
                """
                
                echo "✅ DVWA desplegado en: http://localhost:8082"
            }
        }
    }
    
    post {
        always {
            echo "✅ Pipeline completado"
            echo "📊 SonarQube: http://localhost:9000/dashboard?id=DVWA-Security-App"
            echo "🌐 DVWA: http://localhost:8082"
        }
    }
}
