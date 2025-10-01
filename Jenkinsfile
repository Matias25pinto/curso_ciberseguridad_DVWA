pipeline {
    agent none

    stages {
        stage('Checkout') {
            agent { 
                docker { 
                    image 'php:8.2-cli' 
                } 
            }
            steps {
                sh 'php --version'
                sh 'ls -la'  // Verifica los archivos en workspace
            }
        }

        stage('Compilation') {
            agent { 
                docker { 
                    image 'php:8.2-cli' 
                } 
            }
            steps {
                sh 'echo "Compilando..."'
            }
        }

        stage('Build') {
            agent { 
                docker { 
                    image 'php:8.2-cli' 
                } 
            }
            steps {
                sh 'docker build -t my-php-app .'
            }
        }

        stage('Deploy') {
            agent { 
                docker { 
                    image 'php:8.2-cli' 
                } 
            }
            steps {
                sh 'docker run --rm my-php-app'
            }
        }

        stage('SAST - Semgrep') {
            agent {
                docker {
                    image 'returntocorp/semgrep:latest'
                }
            }
            steps {
                script {
                    // Ejecutar Semgrep y generar reporte JSON
                    sh 'semgrep --config=auto --json > semgrep-report.json || true'

                    // Guardar el reporte en Jenkins
                    archiveArtifacts artifacts: 'semgrep-report.json', fingerprint: true

                    // Leer el reporte y contar findings
                    def report = readJSON file: 'semgrep-report.json'
                    def findings = report.results.size()

                    echo "Se encontraron ${findings} vulnerabilidades en total"

                    // Bloquear pipeline si hay findings high o critical
                    def criticalFindings = report.results.findAll { it.severity in ['high','critical'] }
                    if (criticalFindings.size() > 0) {
                        error("Bloqueando pipeline: vulnerabilidades HIGH o CRITICAL encontradas")
                    }
                }
            }
        }
    }
}

