pipeline {
    agent any
    
    environment {
        IMAGE_NAME = "juice-shop"
        IMAGE_TAG = "latest"
    }
    
    stages {
        stage('Preparación') {
            steps {
                script {
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
        
        stage('Build & Security Scans') {
            steps {
                script {
                    try {
                        // Crear directorios para reportes
                        sh "mkdir -p security-reports"
                        
                        echo "🔨 Construyendo imagen Docker..."
                        sh "docker build -t ${IMAGE_NAME}:${IMAGE_TAG} ."
                        
                        echo "🔍 Ejecutando SAST con Semgrep en código fuente..."
                        sh """
                            docker run --rm \
                                -v "\${PWD}:/src" \
                                --workdir /src \
                                returntocorp/semgrep:latest \
                                semgrep scan --config auto --json --output security-reports/semgrep-report.json || \
                                echo "Semgrep completado con advertencias"
                        """
                        
                        echo "🔍 Analizando dependencias con Trivy (después del build)..."
                        sh """
                            docker run --rm \
                                -v \${PWD}:/workspace \
                                --workdir /workspace \
                                aquasec/trivy:latest \
                                fs . --timeout 5m --scanners vuln \
                                --format json --output security-reports/trivy-fs-report.json || \
                                echo "Trivy FS completado con advertencias"
                        """
                        
                        echo "🔍 Escaneando imagen Docker con Trivy..."
                        sh """
                            docker run --rm \
                                -v /var/run/docker.sock:/var/run/docker.sock \
                                -v \${PWD}/security-reports:/reports \
                                aquasec/trivy:latest \
                                image --timeout 10m \
                                --scanners vuln,secret \
                                --format json --output /reports/trivy-image-report.json \
                                ${IMAGE_NAME}:${IMAGE_TAG} || \
                                echo "Trivy Image completado con advertencias"
                        """
                        
                        echo "🔍 Ejecutando análisis de configuración con Checkov..."
                        sh """
                            docker run --rm \
                                -v \${PWD}:/workspace \
                                --workdir /workspace \
                                bridgecrew/checkov:latest \
                                --directory . \
                                --framework dockerfile \
                                --output json \
                                --output-file-path security-reports/checkov-report.json || \
                                echo "Checkov completado con advertencias"
                        """
                        
                        // Verificar que se generaron los reportes
                        sh "ls -la security-reports/ || echo 'Algunos reportes no se generaron'"
                        
                    } catch (Exception e) {
                        echo "⚠️ Algún scan falló: ${e.getMessage()}"
                        currentBuild.result = 'UNSTABLE'
                    }
                }
            }
        }
        
        stage('DAST - OWASP ZAP') {
            steps {
                script {
                    try {
                        // Crear directorio para reportes ZAP
                        sh "mkdir -p zap-reports"
                        
                        echo "🚀 Iniciando aplicación para DAST..."
                        // Iniciar aplicación en la misma red que Jenkins
                        sh """
                            docker run -d --rm \
                                --name juice-shop-running \
                                --network jenkins_jenkins-network \
                                -p 3000:3000 \
                                ${IMAGE_NAME}:${IMAGE_TAG}
                        """
                        
                        // Esperar a que la aplicación esté lista
                        echo "⏰ Esperando que la aplicación esté lista..."
                        sh "sleep 60"
                        
                        // Verificar logs del contenedor
                        sh "docker logs juice-shop-running | tail -20"
                        
                        // Verificar conectividad usando curl desde Jenkins
                        sh """
                            echo "🔗 Verificando conectividad..."
                            docker run --rm --network jenkins_jenkins-network alpine/curl:latest \
                                curl -f -v http://juice-shop-running:3000 || \
                            (echo '❌ Aplicación no responde' && exit 1)
                        """
                        
                        // Ejecutar ZAP scan con reportes montados
                        echo "🕷️ Ejecutando OWASP ZAP baseline scan..."
                        sh """
                            docker run --rm \
                                --network jenkins_jenkins-network \
                                -v \${PWD}/zap-reports:/zap/wrk/:rw \
                                zaproxy/zap-stable:latest \
                                zap-baseline.py \
                                    -t http://juice-shop-running:3000 \
                                    -d \
                                    -r zap_baseline_report.html \
                                    -J zap_baseline_report.json \
                                    -w zap_baseline_report.md \
                                    -I || echo "ZAP completado con advertencias"
                        """
                        
                        // Verificar que se generaron los reportes
                        sh "ls -la zap-reports/ || echo 'No se generaron reportes ZAP'"
                        
                    } catch (Exception e) {
                        echo "⚠️ DAST falló: ${e.getMessage()}"
                        currentBuild.result = 'UNSTABLE'
                    } finally {
                        // Limpiar recursos
                        sh "docker stop juice-shop-running || true"
                    }
                }
            }
        }
        
        stage('PaC - Checkov (opcional)') {
            steps {
                echo "ℹ️ Checkov ya se ejecutó en el stage 'Build & Security Scans'"
                echo "✅ Análisis de configuración completado"
            }
        }
    }
    
    post {
        always {
            // Limpiar cualquier contenedor que pueda haber quedado
            sh "docker stop juice-shop-running || true"
            sh "docker rm juice-shop-running || true"
            
            // Archivar los artefactos del workspace
            archiveArtifacts artifacts: '**/*', fingerprint: true, allowEmptyArchive: true
            
            echo "📄 Artefactos archivados disponibles en la sección 'Artifacts' de este build"
        }
        
        success {
            echo "🎉 Pipeline completado exitosamente!"
            echo "📄 Artefactos disponibles en la sección 'Artifacts' de este build"
        }
        
        failure {
            echo "❌ Pipeline falló. Revisa los logs para más detalles."
        }
    }
}

