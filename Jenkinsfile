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
                                -v "${WORKSPACE_DIR}:/src" \
                                --workdir /src \
                                returntocorp/semgrep:latest \
                                semgrep scan --config auto --json --output semgrep-report.json . || true
                        """

                        echo "🔍 Escaneando imagen Docker con Trivy..."
                        sh """
                            docker run --rm \
                                -v /var/run/docker.sock:/var/run/docker.sock \
                                -v "${WORKSPACE_DIR}:/workspace" \
                                aquasec/trivy:latest \
                                trivy image --timeout 10m -f json -o trivy-image-report.json \
                                ${IMAGE_NAME}:${IMAGE_TAG} || true
                        """

                        echo "🔍 Ejecutando análisis de configuración con Checkov..."
                        sh """
                            docker run --rm \
                                -v "${WORKSPACE_DIR}:/workspace" \
                                --workdir /workspace \
                                bridgecrew/checkov:latest \
                                checkov --directory . --framework dockerfile -o json --output-file-path . \
                                --soft-fail || true
                        """

                        // Renombrar archivo de Checkov si tiene nombre diferente
                        sh """
                            if [ -f results_json.json ]; then
                                mv results_json.json checkov-report.json
                            fi
                        """

                        echo "✅ Verificando reportes generados..."
                        sh "ls -lh *.json"

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
                                -v "${WORKSPACE_DIR}:/zap/wrk" \
                                zaproxy/zap-stable:latest \
                                zap-baseline.py \
                                    -t http://juice-shop-running:3000 \
                                    -r zap-baseline-report.html \
                                    -J zap-baseline-report.json \
                                    -w zap-baseline-report.md \
                                    -I || true
                        """

                        echo "✅ Verificando reportes ZAP generados..."
                        sh "ls -lh zap-baseline-report.*"

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
                sh "ls -lh *.json || echo 'No hay reportes JSON'"
                sh "ls -lh zap-baseline-report.* || echo 'No hay reportes ZAP'"

                // Archivar todos los reportes JSON y ZAP
                archiveArtifacts artifacts: '*.json', fingerprint: true, allowEmptyArchive: true
                archiveArtifacts artifacts: 'zap-baseline-report.*', fingerprint: true, allowEmptyArchive: true
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