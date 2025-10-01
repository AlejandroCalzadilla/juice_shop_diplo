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
                        // Limpiar reportes antiguos y crear directorios con permisos correctos
                        sh """
                            rm -rf reports/ security-reports/ zap-reports/ 2>/dev/null || true
                            mkdir -p security-reports
                            chmod 777 security-reports
                        """
                        
                        echo "🔨 Construyendo imagen Docker..."
                        sh "docker build -t ${IMAGE_NAME}:${IMAGE_TAG} ."
                        
                        echo "🔍 Ejecutando SAST con Semgrep en código fuente..."
                        sh """
                            docker run --rm \
                                --user \$(id -u):\$(id -g) \
                                -v "\${PWD}:/src" \
                                --workdir /src \
                                returntocorp/semgrep:latest \
                                sh -c "semgrep scan --config auto --json --output security-reports/semgrep-report.json . && \
                                       echo 'Semgrep terminado. Verificando archivo:' && \
                                       ls -la security-reports/semgrep-report.json && \
                                       wc -c security-reports/semgrep-report.json" || \
                                echo "Semgrep completado con advertencias"
                        """
                        
                        echo "🔍 Analizando dependencias con Trivy (después del build)..."
                        sh """
                            docker run --rm \
                                --user \$(id -u):\$(id -g) \
                                -v \${PWD}:/workspace \
                                --workdir /workspace \
                                aquasec/trivy:latest \
                                sh -c "trivy fs . --timeout 5m --scanners vuln \
                                       --format json --output security-reports/trivy-fs-report.json && \
                                       echo 'Trivy FS terminado. Verificando archivo:' && \
                                       ls -la security-reports/trivy-fs-report.json && \
                                       wc -c security-reports/trivy-fs-report.json" || \
                                echo "Trivy FS completado con advertencias"
                        """
                        
                        echo "🔍 Escaneando imagen Docker con Trivy..."
                        sh """
                            docker run --rm \
                                --user \$(id -u):\$(id -g) \
                                -v /var/run/docker.sock:/var/run/docker.sock \
                                -v \${PWD}/security-reports:/reports \
                                aquasec/trivy:latest \
                                sh -c "trivy image --timeout 10m \
                                       --scanners vuln,secret \
                                       --format json --output /reports/trivy-image-report.json \
                                       ${IMAGE_NAME}:${IMAGE_TAG} && \
                                       echo 'Trivy Image terminado. Verificando archivo:' && \
                                       ls -la /reports/trivy-image-report.json && \
                                       wc -c /reports/trivy-image-report.json" || \
                                echo "Trivy Image completado con advertencias"
                        """
                        
                        echo "🔍 Ejecutando análisis de configuración con Checkov..."
                        sh """
                            docker run --rm \
                                --user \$(id -u):\$(id -g) \
                                -v \${PWD}:/workspace \
                                --workdir /workspace \
                                bridgecrew/checkov:latest \
                                sh -c "checkov --directory . \
                                       --framework dockerfile \
                                       --output json \
                                       --output-file-path security-reports/checkov-report.json && \
                                       echo 'Checkov terminado. Verificando archivo:' && \
                                       ls -la security-reports/checkov-report.json && \
                                       wc -c security-reports/checkov-report.json" || \
                                echo "Checkov completado con advertencias"
                        """
                        
                        // Verificar que se generaron los reportes
                        sh """
                            echo "📊 Estado de reportes generados:"
                            echo "================================"
                            ls -la security-reports/ 2>/dev/null || echo "❌ Directorio security-reports no existe"
                            echo ""
                            find . -name "*report*" -type f 2>/dev/null || echo "❌ No se encontraron archivos de reporte"
                            echo ""
                            echo "📁 Contenido completo del workspace:"
                            find . -maxdepth 2 -type f | head -20
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
                        // Crear directorio para reportes ZAP con permisos correctos
                        sh """
                            mkdir -p zap-reports
                            chmod 777 zap-reports
                        """
                        
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
                        
                        // Ejecutar ZAP scan con permisos arreglados
                        echo "🕷️ Ejecutando OWASP ZAP baseline scan..."
                        
                        // Estrategia alternativa: usar un contenedor temporal para copiar archivos
                        sh """
                            # Ejecutar ZAP y mantener el contenedor temporalmente
                            docker run -d --name zap-temp \
                                --network jenkins_jenkins-network \
                                zaproxy/zap-stable:latest \
                                sh -c "zap-baseline.py \
                                    -t http://juice-shop-running:3000 \
                                    -d \
                                    -r /tmp/zap_baseline_report.html \
                                    -J /tmp/zap_baseline_report.json \
                                    -w /tmp/zap_baseline_report.md \
                                    -I; sleep 30" || echo "ZAP iniciado"
                            
                            # Esperar a que termine ZAP
                            sleep 90
                            
                            # Verificar logs de ZAP
                            echo "📋 Logs de ZAP:"
                            docker logs zap-temp | tail -20
                            
                            # Copiar archivos del contenedor al workspace
                            echo "� Copiando archivos de ZAP al workspace:"
                            docker cp zap-temp:/tmp/zap_baseline_report.html zap-reports/zap_baseline_report.html 2>/dev/null || echo "HTML no copiado"
                            docker cp zap-temp:/tmp/zap_baseline_report.json zap-reports/zap_baseline_report.json 2>/dev/null || echo "JSON no copiado"
                            docker cp zap-temp:/tmp/zap_baseline_report.md zap-reports/zap_baseline_report.md 2>/dev/null || echo "MD no copiado"
                            
                            # Limpiar contenedor temporal
                            docker stop zap-temp 2>/dev/null || true
                            docker rm zap-temp 2>/dev/null || true
                            
                            # Verificar archivos copiados
                            echo "✅ Archivos copiados al workspace:"
                            ls -la zap-reports/
                        """
                        
                        // Verificar que se generaron los reportes ZAP
                        sh """
                            echo "🕷️ Estado final de reportes ZAP:"
                            echo "================================"
                            if [ -d "zap-reports" ] && [ "\$(ls -A zap-reports 2>/dev/null)" ]; then
                                echo "✅ Directorio zap-reports contiene archivos:"
                                ls -la zap-reports/
                                echo ""
                                echo "� Tamaños de archivos ZAP:"
                                find zap-reports -name "*.html" -o -name "*.json" -o -name "*.md" | xargs wc -c 2>/dev/null || echo "Archivos no encontrados"
                            else
                                echo "❌ Directorio zap-reports está vacío o no existe"
                                echo "🔍 Búsqueda alternativa de archivos ZAP:"
                                find . -name "*zap_baseline*" -type f 2>/dev/null || echo "No se encontraron archivos ZAP en el workspace"
                            fi
                        """
                        
                    } catch (Exception e) {
                        echo "⚠️ DAST falló: ${e.getMessage()}"
                        currentBuild.result = 'UNSTABLE'
                    } finally {
                        // Limpiar recursos
                        sh "docker stop juice-shop-running || true"
                        sh "docker stop zap-temp || true"
                        sh "docker rm zap-temp || true"
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
            
            // Mostrar resumen final de todos los archivos generados
            sh """
                echo "📋 RESUMEN FINAL DE REPORTES GENERADOS:"
                echo "======================================"
                echo ""
                echo "🔍 Buscando todos los reportes..."
                find . -name "*.json" -o -name "*.html" -o -name "*.md" | grep -E "(report|zap)" || echo "❌ No se encontraron reportes"
                echo ""
                echo "📁 Estructura de directorios:"
                ls -la security-reports/ 2>/dev/null || echo "❌ security-reports/ no existe"
                ls -la zap-reports/ 2>/dev/null || echo "❌ zap-reports/ no existe"
                echo ""
                echo "📊 Tamaño de archivos de reporte:"
                find . -name "*report*" -type f -exec ls -lh {} \\; 2>/dev/null || echo "❌ No hay archivos de reporte"
            """
            
            // Archivar específicamente los reportes de seguridad (usando patrones más simples)
            script {
                try {
                    // Verificar si existen archivos antes de archivar
                    def hasSecurityReports = sh(script: "ls security-reports/*.json 2>/dev/null", returnStatus: true) == 0
                    def hasZapReports = sh(script: "ls zap-reports/* 2>/dev/null", returnStatus: true) == 0
                    
                    if (hasSecurityReports) {
                        archiveArtifacts artifacts: 'security-reports/*', fingerprint: true, allowEmptyArchive: true
                        echo "✅ Reportes de seguridad archivados"
                    } else {
                        echo "❌ No se encontraron reportes de seguridad para archivar"
                    }
                    
                    if (hasZapReports) {
                        archiveArtifacts artifacts: 'zap-reports/*', fingerprint: true, allowEmptyArchive: true
                        echo "✅ Reportes ZAP archivados"
                    } else {
                        echo "❌ No se encontraron reportes ZAP para archivar"
                    }
                    
                    // Archivar archivos importantes del proyecto
                    archiveArtifacts artifacts: 'Dockerfile, package.json, *.md', fingerprint: true, allowEmptyArchive: true
                    
                } catch (Exception e) {
                    echo "⚠️ Error archivando algunos artefactos: ${e.getMessage()}"
                }
            }
            
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

