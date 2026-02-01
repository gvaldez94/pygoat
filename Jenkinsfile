pipeline {
    agent any

    environment {
        // Credenciales configuradas en Administrar Jenkins > Credentials
        DOJO_TOKEN = credentials('DEFECTDOJO_TOKEN')
        DT_TOKEN   = credentials('DTRACK_TOKEN')

        // URLs de red interna de Docker
        DOJO_URL   = 'http://django-defectdojo-nginx-1:8080'
        DT_URL     = 'http://dependency-track-apiserver-1:8080'
        
        // ID del Engagement de DefectDojo
        ENGAGEMENT_ID = '1'
        
        DOCKER_TLS_VERIFY = ""  // Esto soluciona el error del ca.pem
        DOCKER_CERT_PATH = ""
        DOCKER_HOST = "unix:///var/run/docker.sock" // Asegura que use el socket
    }

    stages {
        stage ('Check Docker') {
            steps {
                sh 'unset DOCKER_TLS_VERIFY && unset DOCKER_CERT_PATH && docker ps'
            }
        }
        
        stage('Checkout') {
            steps {
                git branch: 'master', url: 'https://github.com/gvaldez94/pygoat.git'
            }
        }

        stage('SAST - Bandit') {
            steps {
                script {
                    // Usamos el nombre del volumen físico 'jenkins-data'
                    // Y le indicamos la ruta relativa dentro de ese volumen
                    def relativePath = env.WORKSPACE.replace("/var/jenkins_home/", "")
                    
                    sh """
                        docker run --rm -u 0:0 \
                        -v jenkins-data:/var/jenkins_home \
                        cytopia/bandit -r /var/jenkins_home/${relativePath} -f json -o /var/jenkins_home/${relativePath}/bandit-report.json || true
                    """
                    
                    // El "Security Gate" que ahora no detendrá el pipeline
                    catchError(buildResult: 'SUCCESS', stageResult: 'UNSTABLE') {
                        echo "Ejecutando Security Gate para Bandit..."
                        sh """
                            docker run --rm -u 0:0 \
                            -v jenkins-data:/var/jenkins_home \
                            cytopia/bandit -r /var/jenkins_home/${relativePath} -lll
                        """
                    }
                }
            }
        }

        stage('SCA - Dependency-Track') {
            steps {
                script {
                    def relativePath = env.WORKSPACE.replace("/var/jenkins_home/", "")
                    
                    sh """
                        docker run --rm -u 0:0 \
                        -v jenkins-data:/var/jenkins_home \
                        cyclonedx/cyclonedx-python:latest requirements /var/jenkins_home/${relativePath}/requirements.txt -o /var/jenkins_home/${relativePath}/bom.json
                    """
                    
                    echo "Subiendo BOM a Dependency-Track..."
                    sh "curl -X POST '${DT_URL}/api/v1/bom' -H 'X-Api-Key: ${DT_TOKEN}' -F 'projectName=PyGoat' -F 'projectVersion=1.0' -F 'autoCreate=true' -F 'bom=@bom.json'"
                }
            }
        }

        stage('Secrets - Gitleaks') {
            steps {
                script {
                    // Usamos la carpeta local mapeada directamente para evitar líos de rutas
                    sh """
                        docker run --rm -v ${env.WORKSPACE}:/src zricethezav/gitleaks:latest detect \
                        --source=/src --report-path=/src/gitleaks-report.json --exit-code 0 || true
                    """
                }
            }
        }

        stage('Integración DefectDojo') {
            steps {
                script {
                    def engagementID = "1"
                    
                    // IMPORTANTE: Listamos los archivos para confirmar que existen antes del curl
                    sh "ls -lh bandit-report.json gitleaks-report.json || echo 'Archivos no encontrados'"

                    echo "Subiendo hallazgos de Bandit a DefectDojo (Engagement: ${engagementID})..."
                    sh """
                        curl -X POST "http://django-defectdojo-nginx-1:8080/api/v2/import-scan/" \
                        -H "Authorization: Token ${DOJO_TOKEN}" \
                        -F "active=true" \
                        -F "verified=false" \
                        -F "scan_type=Bandit Scan" \
                        -F "engagement=${engagementID}" \
                        -F "file=@bandit-report.json"
                    """
                    
                    echo "Subiendo hallazgos de Dependency Track a DefectDojo..."
                    sh """
                        curl -X POST "http://django-defectdojo-nginx-1:8080/api/v2/import-scan/" \
                        -H "Authorization: Token ${DOJO_TOKEN}" \
                        -F "active=true" \
                        -F "verified=false" \
                        -F "scan_type=CycloneDX Scan" \
                        -F "engagement=${engagementID}" \
                        -F "file=@bom.json"
                    """

                    echo "Subiendo hallazgos de Gitleaks a DefectDojo..."
                    sh """
                        curl -X POST "http://django-defectdojo-nginx-1:8080/api/v2/import-scan/" \
                        -H "Authorization: Token ${DOJO_TOKEN}" \
                        -F "active=true" \
                        -F "verified=false" \
                        -F "scan_type=Gitleaks Scan" \
                        -F "engagement=${engagementID}" \
                        -F "file=@gitleaks-report.json"
                    """
                }
            }
        }
    }
}