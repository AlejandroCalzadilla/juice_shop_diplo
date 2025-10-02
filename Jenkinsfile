pipeline {
    agent any
    
    environment {
        IMAGE_NAME = "juice-shop"
        IMAGE_TAG = "latest"
        WORKSPACE_DIR = "${WORKSPACE}"
        JENKINS_UID = "1000"
        JENKINS_GID = "1000"
    }
    
    stages {
        stage('Preparación') {
            steps {
                script {
                    // Obtener UID y GID reales de Jenkins
                    sh '''
                        echo "🔍 Usuario Jenkins info:"
                        id
                        JENKINS_UID=$(id -u)
                        JENKINS_GID=$(id -g)
                        echo "Jenkins UID: $JENKINS_UID, GID: $JENKINS_GID"
                    '''
                    
                    // Limpiar contenedores anteriores
                    sh "docker stop juice-shop-running || true"
                    sh "docker rm juice-shop-running || true"
                    sh "docker rmi ${IMAGE_NAME}:${IMAGE_TAG} || true"
                }
            }
        }
        
        stage('Checkout') {
            steps {
                git branch: 'main', url: 'https://github.com/AlejandroCalzadilla/juice_shop_diplo.git'
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
                                --workdir /workspace \
                                aquasec/trivy:latest \
                                image --format json ${IMAGE_NAME}:${IMAGE_TAG} > trivy-image-report.json || true
                        """

                        echo "🔍 Ejecutando análisis de configuración con Checkov..."
                        sh """
                            docker run --rm \
                                -v "${WORKSPACE_DIR}:/workspace" \
                                --workdir /workspace \
                                bridgecrew/checkov:latest \
                                --directory . --framework dockerfile -o json --output-file-path . \
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
                            # Crear directorio ZAP con permisos correctos
                            mkdir -p security-reports/zap
                            chmod 755 security-reports/zap
                            
                            # Ejecutar ZAP con usuario correcto
                            docker run --rm \
                                --user \$(id -u):\$(id -g) \
                                --network jenkins_jenkins-network \
                                -v "${WORKSPACE_DIR}/security-reports/zap:/zap/wrk" \
                                zaproxy/zap-stable:latest \
                                sh -c "zap-baseline.py \
                                    -t http://juice-shop-running:3000 \
                                    -J /zap/wrk/zap-baseline-report.json \
                                    -I && \
                                    chmod 644 /zap/wrk/zap-baseline-report.json" || \
                                echo "ZAP falló, creando reporte vacío" && \
                                echo '{"errors":["ZAP failed"}' > security-reports/zap/zap-baseline-report.json
                        """

                        echo "✅ Verificando reportes ZAP generados..."
                        sh """
                            echo "📊 Reportes ZAP:"
                            ls -lh security-reports/zap/ || echo "Directorio ZAP vacío"
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

                echo "📋 RESUMEN FINAL DE REPORTES:"
                echo "============================="
                sh """
                    echo "🔍 Todos los archivos JSON generados:"
                    ls -lh *.json || echo "No hay reportes"
                    
                    echo ""
                    echo "📊 Contenido de reportes (primeras líneas):"
                    for file in *.json; do
                        if [ -f "\$file" ]; then
                            echo "=== \$file ==="
                            head -3 "\$file"
                        fi
                    done || echo "No se pueden leer reportes"
                """

                // Archivar todos los reportes de seguridad
                archiveArtifacts artifacts: '*.json', fingerprint: true, allowEmptyArchive: true
                
                // También archivar archivos importantes del proyecto
                archiveArtifacts artifacts: 'Dockerfile, package.json, *.md', fingerprint: true, allowEmptyArchive: true
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