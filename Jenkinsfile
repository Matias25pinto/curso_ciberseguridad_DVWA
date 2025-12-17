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
                                
                                # 3. Obtener reporte JSON via API
                                curl -s -u ${SONAR_TOKEN}: \
                                  "http://sonarqube:9000/api/issues/search?componentKeys=DVWA-Security-App&resolved=false&ps=1000" \
                                  -o ../sonarqube.json
                                
                                # 4. Mover al directorio principal
                                mv ../sonarqube.json . 2>/dev/null || true
                            '''
                        } catch (err) {
                            unstable(message: "SonarQube encontró hallazgos de seguridad")
                        }
                    }
                    
                    // Verificar que el archivo se creó
                    sh '''
                        test -f dvwa/sonarqube.json && mv dvwa/sonarqube.json . || echo "Archivo no existe, creando vacío..."
                        test -f sonarqube.json || echo "{}" > sonarqube.json
                        echo "=== Archivo sonarqube.json ==="
                        ls -la sonarqube.json
                    '''
                }
                
                // Archivar resultados
                archiveArtifacts artifacts: 'sonarqube.json', fingerprint: true, allowEmptyArchive: true
            }
            
            post {
                always {
                    script {
                        if (fileExists('sonarqube.json')) {
                            try {
                                def results = readJSON file: 'sonarqube.json'
                                echo "SonarQube encontró ${results.total ?: 0} issues"
                                
                                // Mostrar tipos de issues
                                if (results.issues) {
                                    def bugs = results.issues.count { it.type == 'BUG' }
                                    def vulns = results.issues.count { it.type == 'VULNERABILITY' }
                                    def smells = results.issues.count { it.type == 'CODE_SMELL' }
                                    
                                    echo "  - Bugs: ${bugs}"
                                    echo "  - Vulnerabilidades: ${vulns}"
                                    echo "  - Code Smells: ${smells}"
                                }
                            } catch (Exception e) {
                                echo "No se pudo procesar el JSON: ${e.message}"
                            }
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
}
