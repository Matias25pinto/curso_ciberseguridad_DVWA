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
                    // Recuperar el código stasheado
                    unstash 'dvwa-code'
                    
                    // Eliminar archivo anterior si existe
                    sh 'rm -f sonarqube.json || true'
                    
                    withCredentials([string(credentialsId: 'sonarqube-token', variable: 'SONAR_TOKEN')]) {
                        try {
                            // Ejecutar SonarQube y obtener reporte JSON
                            sh '''
                                cd dvwa
                                
                                # 1. Ejecutar análisis
                                sonar-scanner \
                                  -Dsonar.projectKey=DVWA-Security-App \
                                  -Dsonar.sources=. \
                                  -Dsonar.host.url=http://sonarqube:9000 \
                                  -Dsonar.token=${SONAR_TOKEN}
                                
                                # 2. Esperar a que se procese
                                sleep 10
                                
                                # 3. Obtener reporte JSON via API (usando ps=500 que es el máximo permitido)
                                curl -s -u ${SONAR_TOKEN}: \
                                  "http://sonarqube:9000/api/issues/search?componentKeys=DVWA-Security-App&resolved=false&ps=500" \
                                  -o ../sonarqube.json
                                
                                # 4. Mover al directorio principal
                                mv ../sonarqube.json . 2>/dev/null || true
                            '''
                        } catch (err) {
                            error("CRÍTICO: Se encontraron vulnerabilidades de nivel ERROR. Pipeline abortado.")
                        }
                    }
                    
                    // Verificar que el archivo se creó y mostrar contenido
                    sh '''
                        test -f dvwa/sonarqube.json && mv dvwa/sonarqube.json . || echo "Archivo no existe, creando vacío..."
                        test -f sonarqube.json || echo "{}" > sonarqube.json
                        echo "=== Archivo sonarqube.json ==="
                        ls -la sonarqube.json
                        
                        # Mostrar contenido del archivo
                        echo "=== Contenido del JSON ==="
                        cat sonarqube.json || echo "No se pudo leer el archivo"
                    '''
                }
                
                // Archivar resultados
                archiveArtifacts artifacts: 'sonarqube.json', fingerprint: true, allowEmptyArchive: true
            }
            
            post {
                always {
                    echo "✅ Análisis de SonarQube completado"
                    echo "📊 Dashboard: http://localhost:9000/dashboard?id=DVWA-Security-App"
                    echo "📄 Reporte JSON archivado como sonarqube.json"
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
