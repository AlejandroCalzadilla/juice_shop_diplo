pipeline {
    agent any
    
    environment {
        IMAGE_NAME = "juice-shop"
        IMAGE_TAG = "latest"
        WORKSPACE_DIR = "${WORKSPACE}"
    }
    
    stages {
        stage('Preparación') {
            steps {
                script {
                    // Limpiar contenedores anteriores
                    sh "docker stop juice-shop-running || true"
                    sh "docker rm juice-shop-running || true"
                    
                    // Limpiar imágenes anteriores
                    sh "docker rmi ${IMAGE_NAME}:${IMAGE_TAG} || true"
                    
                    // Crear directorios de reportes con permisos correctos
                    sh """
                        rm -rf security-reports/ zap-reports/ 2>/dev/null || true
                        mkdir -p security-reports zap-reports
                        chmod -R 777 security-reports zap-reports
                    """
                }
            }
        }
        
        stage('Checkout') {
            steps {
                git branch: 'main', url: 'https://github.com/juice-shop/juice-shop.git'
            }
        }
        
        stage('Build & Security Scans') {
            steps {
                script {
                    try {
                        echo "🔨 Construyendo imagen Docker..."
                        sh "docker build -t ${IMAGE_NAME}:${IMAGE_TAG} ."
                        
                        echo "🔍 Ejecutando SAST con Semgrep..."
                        sh """
                            docker run --rm \
                                -v "${WORKSPACE_DIR}:/src:ro" \
                                -v "${WORKSPACE_DIR}/security-reports:/reports:rw" \
                                --workdir /src \
                                returntocorp/semgrep:latest \
                                semgrep scan --config auto --json --output /reports/semgrep-report.json . || true
                        """
                        
                        echo "🔍 Analizando dependencias con Trivy (filesystem)..."
                        sh """
                            docker run --rm \
                                -v "${WORKSPACE_DIR}:/workspace:ro" \
                                -v "${WORKSPACE_DIR}/security-reports:/reports:rw" \
                                --workdir /workspace \
                                aquasec/trivy:latest \
                                trivy fs . --timeout 5m --scanners vuln \
                                    --format json --output /reports/trivy-fs-report.json || true
                        """
                        
                        echo "🔍 Escaneando imagen Docker con Trivy..."
                        sh """
                            docker run --rm \
                                -v /var/run/docker.sock:/var/run/docker.sock:ro \
                                -v "${WORKSPACE_DIR}/security-reports:/reports:rw" \
                                aquasec/trivy:latest \
                                trivy image --timeout 10m --scanners vuln,secret \
                                    --format json --output /reports/trivy-image-report.json \
                                    ${IMAGE_NAME}:${IMAGE_TAG} || true
                        """
                        
                        echo "🔍 Ejecutando análisis de configuración con Checkov..."
                        sh """
                            docker run --rm \
                                -v "${WORKSPACE_DIR}:/workspace:ro" \
                                -v "${WORKSPACE_DIR}/security-reports:/reports:rw" \
                                --workdir /workspace \
                                bridgecrew/checkov:latest \
                                checkov --directory . --framework dockerfile \
                                    --output json --output-file-path /reports \
                                    --soft-fail || true
                        """
                        
                        // Renombrar archivo de Checkov si tiene nombre diferente
                        sh """
                            if [ -f security-reports/results_json.json ]; then
                                mv security-reports/results_json.json security-reports/checkov-report.json
                            fi
                        """
                        
                        echo "✅ Verificando reportes generados..."
                        sh """
                            echo "📊 Reportes en security-reports/:"
                            ls -lh security-reports/
                        """
                        
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
                        echo "🚀 Iniciando aplicación para DAST..."
                        sh """
                            docker run -d --rm \
                                --name juice-shop-running \
                                --network jenkins_jenkins-network \
                                -p 3000:3000 \
                                ${IMAGE_NAME}:${IMAGE_TAG}
                        """
                        
                        echo "⏰ Esperando que la aplicación esté lista..."
                        sh "sleep 60"
                        
                        echo "🔗 Verificando conectividad..."
                        sh """
                            docker run --rm --network jenkins_jenkins-network alpine/curl:latest \
                                curl -f -v http://juice-shop-running:3000 || true
                        """
                        
                        echo "🕷️ Ejecutando OWASP ZAP baseline scan..."
                        sh """
                            docker run --rm \
                                --network jenkins_jenkins-network \
                                -v "${WORKSPACE_DIR}/zap-reports:/zap/wrk:rw" \
                                zaproxy/zap-stable:latest \
                                zap-baseline.py \
                                    -t http://juice-shop-running:3000 \
                                    -r zap-baseline-report.html \
                                    -J zap-baseline-report.json \
                                    -w zap-baseline-report.md \
                                    -I || true
                        """
                        
                        echo "✅ Verificando reportes ZAP generados..."
                        sh """
                            echo "📊 Reportes en zap-reports/:"
                            ls -lh zap-reports/
                        """
                        
                    } catch (Exception e) {
                        echo "⚠️ DAST falló: ${e.getMessage()}"
                        currentBuild.result = 'UNSTABLE'
                    } finally {
                        sh "docker stop juice-shop-running || true"
                    }
                }
            }
        }
    }
    
    post {
        always {
            script {
                // Limpiar contenedores
                sh "docker stop juice-shop-running || true"
                sh "docker rm juice-shop-running || true"
                
                echo "📋 RESUMEN DE REPORTES GENERADOS:"
                echo "=================================="
                
                // Verificar y mostrar reportes de seguridad
                sh """
                    echo ""
                    echo "🔒 Reportes de Seguridad (SAST/SCA):"
                    ls -lh security-reports/ 2>/dev/null || echo "❌ No hay reportes de seguridad"
                """
                
                // Verificar y mostrar reportes ZAP
                sh """
                    echo ""
                    echo "🕷️ Reportes DAST (ZAP):"
                    ls -lh zap-reports/ 2>/dev/null || echo "❌ No hay reportes ZAP"
                """
                
                // Archivar SOLO los reportes de seguridad
                try {
                    def securityFiles = sh(script: "ls security-reports/ 2>/dev/null | wc -l", returnStdout: true).trim()
                    def zapFiles = sh(script: "ls zap-reports/ 2>/dev/null | wc -l", returnStdout: true).trim()
                    
                    if (securityFiles.toInteger() > 0) {
                        archiveArtifacts artifacts: 'security-reports/**/*', 
                                       fingerprint: true, 
                                       allowEmptyArchive: false
                        echo "✅ ${securityFiles} reportes de seguridad archivados"
                    } else {
                        echo "⚠️ No se encontraron reportes de seguridad para archivar"
                    }
                    
                    if (zapFiles.toInteger() > 0) {
                        archiveArtifacts artifacts: 'zap-reports/**/*', 
                                       fingerprint: true, 
                                       allowEmptyArchive: false
                        echo "✅ ${zapFiles} reportes ZAP archivados"
                    } else {
                        echo "⚠️ No se encontraron reportes ZAP para archivar"
                    }
                    
                    echo ""
                    echo "📦 Total de artefactos generados: ${securityFiles.toInteger() + zapFiles.toInteger()}"
                    
                } catch (Exception e) {
                    echo "⚠️ Error archivando artefactos: ${e.getMessage()}"
                }
            }
        }
        
        success {
            echo "🎉 Pipeline completado exitosamente!"
            echo "📄 Todos los reportes están disponibles en la sección 'Artifacts'"
        }
        
        failure {
            echo "❌ Pipeline falló. Revisa los logs para más detalles."
        }
        
        unstable {
            echo "⚠️ Pipeline completado con advertencias."
            echo "📄 Los reportes disponibles están en la sección 'Artifacts'"
        }
    }
}