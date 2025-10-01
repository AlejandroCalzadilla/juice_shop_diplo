pipeline {
    agent any
    
    environment {
        // Variables para los reportes
        REPORTS_DIR = "${WORKSPACE}/reports"
        IMAGE_NAME = "juice-shop"
        IMAGE_TAG = "latest"
    }
    
    stages {
        stage('Preparación') {
            steps {
                script {
                    // Crear directorio para reportes
                    sh "mkdir -p ${REPORTS_DIR}"
                    
                    // Limpiar contenedores anteriores si existen
                    sh "docker stop juice-shop-running || true"
                    sh "docker rm juice-shop-running || true"
                    
                    // Limpiar imágenes anteriores
                    sh "docker rmi ${IMAGE_NAME}:${IMAGE_TAG} || true"
                }
            }
        }
        
        stage('Checkout') {
            steps {
                // Cambiar por tu repositorio
                git branch: 'main', url: 'https://github.com/juice-shop/juice-shop.git'
            }
        }
        
        stage('SAST - Semgrep') {
            steps {
                script {
                    try {
                        sh """
                            docker run --rm \
                                -v "\${PWD}:/src" \
                                returntocorp/semgrep \
                                semgrep scan --config auto --json \
                                -o /src/reports/sast-report.json
                        """
                    } catch (Exception e) {
                        echo "⚠️ Semgrep falló, continuando: ${e.getMessage()}"
                        sh "echo '{}' > ${REPORTS_DIR}/sast-report.json"
                    }
                }
            }
        }
        
        stage('SCA - Trivy Filesystem') {
            steps {
                script {
                    try {
                        sh """
                            docker run --rm \
                                -v \${PWD}:/app \
                                aquasec/trivy:latest \
                                fs /app --format json \
                                --output /app/reports/sca-report.json
                        """
                    } catch (Exception e) {
                        echo "⚠️ Trivy SCA falló, continuando: ${e.getMessage()}"
                        sh "echo '{}' > ${REPORTS_DIR}/sca-report.json"
                    }
                }
            }
        }
        
        stage('Build & Image Scan') {
            steps {
                script {
                    try {
                        // Construir imagen
                        sh "docker build -t ${IMAGE_NAME}:${IMAGE_TAG} ."
                        
                        // Escanear imagen directamente sin usar /tmp
                        sh """
                            docker run --rm \
                                -v /var/run/docker.sock:/var/run/docker.sock \
                                -v \${PWD}/reports:/reports \
                                aquasec/trivy:latest \
                                image --format json \
                                --output /reports/image-report.json \
                                ${IMAGE_NAME}:${IMAGE_TAG}
                        """
                        
                    } catch (Exception e) {
                        echo "⚠️ Build o Image Scan falló: ${e.getMessage()}"
                        sh "echo '{}' > ${REPORTS_DIR}/image-report.json"
                    }
                }
            }
        }
        
        stage('DAST - OWASP ZAP') {
            steps {
                script {
                    try {
                        // Iniciar aplicación en background
                        sh """
                            docker run -d --rm \
                                --name juice-shop-running \
                                -p 3000:3000 \
                                ${IMAGE_NAME}:${IMAGE_TAG}
                        """
                        
                        // Esperar más tiempo y verificar logs
                        sh "sleep 120"
                        
                        // Verificar logs del contenedor
                        sh "docker logs juice-shop-running"
                        
                        // Verificar que la aplicación responda con mejor diagnóstico
                        sh """
                            echo "Verificando conectividad..."
                            docker exec jenkins-server curl -f http://localhost:3000 || \
                            docker exec juice-shop-running curl -f http://localhost:3000 || \
                            (echo 'Aplicación no responde después de 60s' && exit 1)
                        """
                        
                        // Ejecutar ZAP scan usando la red del contenedor
                        sh """
                            docker run --rm \
                                --link juice-shop-running:target \
                                -v \${PWD}/reports:/zap/wrk/:rw \
                                owasp/zap2docker-stable \
                                zap-baseline.py \
                                -t http://target:3000 \
                                -J dast-report.json || true
                        """
                        
                    } catch (Exception e) {
                        echo "⚠️ DAST falló: ${e.getMessage()}"
                        sh "echo '{}' > ${REPORTS_DIR}/dast-report.json"
                    } finally {
                        // Detener contenedor de la aplicación
                        sh "docker stop juice-shop-running || true"
                    }
                }
            }
        }
        
        stage('PaC - Checkov (opcional)') {
            steps {
                script {
                    try {
                        sh """
                            docker run --rm \
                                -v \${PWD}:/project \
                                bridgecrew/checkov \
                                --directory /project \
                                --output json \
                                --output-file-path /project/reports/pac-report.json || true
                        """
                    } catch (Exception e) {
                        echo "⚠️ Checkov falló: ${e.getMessage()}"
                        sh "echo '{}' > ${REPORTS_DIR}/pac-report.json"
                    }
                }
            }
        }
    }
    
    post {
        always {
            // Limpiar contenedores
            sh "docker stop juice-shop-running || true"
            sh "docker rm juice-shop-running || true"
            
            // Archivar reportes
            archiveArtifacts artifacts: 'reports/*.json', fingerprint: true, allowEmptyArchive: true
            
            // Mostrar resumen de archivos generados
            sh "ls -la ${REPORTS_DIR}/ || echo 'No hay reportes'"
        }
        
        success {
            echo "🎉 Pipeline completado exitosamente!"
        }
        
        failure {
            echo "❌ Pipeline falló. Revisa los logs para más detalles."
        }
    }
}