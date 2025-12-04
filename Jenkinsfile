pipeline {
    agent any

    stages {

        stage('Checkout') {
            steps {
                echo "Obteniendo el código desde GitHub..."
                sh 'rm -rf dvwa || true'
                sh 'git clone https://github.com/Matias25pinto/curso_ciberseguridad_DVWA dvwa'
            }
        }

        stage('Compilation') {
            steps {
                echo "Simulando compilación (PHP no compila, pero hacemos lint)..."
                sh 'cd dvwa && php -l index.php || true'
                sh 'echo "Compilación simulada completada"'
            }
        }

        stage('Building') {
            steps {
                echo "Construyendo Docker image para DVWA..."
                sh 'cd dvwa && docker build -t dvwa-app:latest .'
            }
        }

        stage('Push to Registry (optional)') {
            when {
                expression { false } // desactivado por ahora
            }
          //ejemplo de como usar creadenciales de jenkins
            steps {
                withCredentials([usernamePassword(credentialsId: 'docker-cred', passwordVariable: 'PASS', usernameVariable: 'USER')]) {
                    sh """
                        echo $PASS | docker login -u $USER --password-stdin
                        docker push mi-app:latest
                    """
                }
            }
        }

        stage('Deploy') {
            steps {
                echo "Desplegando DVWA con Docker..."

                sh """
                    docker rm -f dvwa-app || true
                    docker run -d --name dvwa-app -p 8082:80 dvwa-app:latest
                """
            }
        }
    }
}
