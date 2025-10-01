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
                                --user root \
                                -v "\${PWD}:/src" \
                                --workdir /src \
                                returntocorp/semgrep:latest \
                                semgrep scan --config auto --json \
                                --output reports/sast-report.json \
                                --verbose
                        """
                        // Verificar que el archivo se creó correctamente
                        sh "ls -la reports/sast-report.json && wc -c reports/sast-report.json"
                    } catch (Exception e) {
                        echo "⚠️ Semgrep falló, continuando: ${e.getMessage()}"
                        sh "echo '{\"error\": \"SAST failed: ${e.getMessage()}\"}' > ${REPORTS_DIR}/sast-report.json"
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
                                --user root \
                                -v \${PWD}:/workspace \
                                --workdir /workspace \
                                aquasec/trivy:latest \
                                fs . --format json \
                                --output reports/sca-report.json \
                                --timeout 5m
                        """
                        // Verificar que el archivo se creó correctamente  
                        sh "ls -la reports/sca-report.json && wc -c reports/sca-report.json"
                    } catch (Exception e) {
                        echo "⚠️ Trivy SCA falló, continuando: ${e.getMessage()}"
                        sh "echo '{\"error\": \"SCA failed: ${e.getMessage()}\"}' > ${REPORTS_DIR}/sca-report.json"
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
                        
                        // Escanear imagen con output directo al workspace
                        sh """
                            docker run --rm \
                                --user root \
                                -v /var/run/docker.sock:/var/run/docker.sock \
                                -v \${PWD}:/workspace \
                                --workdir /workspace \
                                aquasec/trivy:latest \
                                image --format json \
                                --output reports/image-report.json \
                                --timeout 10m \
                                ${IMAGE_NAME}:${IMAGE_TAG}
                        """
                        // Verificar que el archivo se creó correctamente
                        sh "ls -la reports/image-report.json && wc -c reports/image-report.json"
                        
                    } catch (Exception e) {
                        echo "⚠️ Build o Image Scan falló: ${e.getMessage()}"
                        sh "echo '{\"error\": \"Image scan failed: ${e.getMessage()}\"}' > ${REPORTS_DIR}/image-report.json"
                    }
                }
            }
        }
        
        stage('DAST - OWASP ZAP') {
            steps {
                script {
                    try {
                        // Iniciar aplicación en la misma red que Jenkins
                        sh """
                            docker run -d --rm \
                                --name juice-shop-running \
                                --network jenkins_jenkins-network \
                                -p 3000:3000 \
                                ${IMAGE_NAME}:${IMAGE_TAG}
                        """
                        
                        // Esperar a que la aplicación esté lista
                        sh "sleep 60"
                        
                        // Verificar logs del contenedor
                        sh "docker logs juice-shop-running"
                        
                        // Verificar conectividad usando curl desde Jenkins
                        sh """
                            echo "Verificando conectividad..."
                            docker run --rm --network jenkins_jenkins-network alpine/curl:latest \
                                curl -f http://juice-shop-running:3000 || \
                            (echo 'Aplicación no responde' && exit 1)
                        """
                        
                        // Ejecutar ZAP scan en la misma red de Jenkins
                        sh """
                            docker run --rm \
                                --network jenkins_jenkins-network \
                                --user root \
                                -v \${PWD}/reports:/zap/wrk/:rw \
                                zaproxy/zap-stable:latest \
                                zap-baseline.py \
                                -t http://juice-shop-running:3000 \
                                -J dast-report.json \
                                -I || true
                        """
                        
                        // Verificar que el archivo se creó correctamente
                        sh "ls -la reports/dast-report.json && wc -c reports/dast-report.json || echo 'DAST report not found'"
                        
                    } catch (Exception e) {
                        echo "⚠️ DAST falló: ${e.getMessage()}"
                        sh "echo '{\"error\": \"DAST failed\"}' > ${REPORTS_DIR}/dast-report.json"
                    } finally {
                        // Limpiar recursos
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
                                -v \${PWD}:/workspace \
                                --workdir /workspace \
                                bridgecrew/checkov:latest \
                                --directory . \
                                --output json \
                                --output-file-path reports/pac-report.json || true
                        """
                    } catch (Exception e) {
                        echo "⚠️ Checkov falló: ${e.getMessage()}"
                        sh "echo '{\"error\": \"PaC scan failed\"}' > ${REPORTS_DIR}/pac-report.json"
                    }
                }
            }
        }
    }
    
    post {
        always {
            // Limpiar cualquier contenedor que pueda haber quedado
            sh "docker stop juice-shop-running || true"
            sh "docker rm juice-shop-running || true"
            
            // Archivar reportes
            archiveArtifacts artifacts: 'reports/*.json', fingerprint: true, allowEmptyArchive: true
            
            // Mostrar resumen de archivos generados
            sh "ls -la ${REPORTS_DIR}/ || echo 'No hay reportes'"
            
            // Mostrar tamaño de los reportes para verificar contenido
            sh "find ${REPORTS_DIR} -name '*.json' -exec wc -c {} + || true"
            
            // Mostrar contenido de los reportes para debug
            sh "echo '=== CONTENIDO DE REPORTES ===' && cat ${REPORTS_DIR}/*.json | head -50 || true"
        }
        
        success {
            echo "🎉 Pipeline completado exitosamente!"
        }
        
        failure {
            echo "❌ Pipeline falló. Revisa los logs para más detalles."
        }
    }
}