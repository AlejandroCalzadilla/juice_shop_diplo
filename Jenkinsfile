pipeline {
    agent any
    
    environment {
        IMAGE_NAME = "juice-shop"
        IMAGE_TAG = "latest"
        WORKSPACE_DIR = "${WORKSPACE}"
        JENKINS_UID = "1000"
        JENKINS_GID = "1000"
        // Configurar permisos máximos
        UMASK = "000"
        // Variables para asegurar acceso total
        HOME = "${WORKSPACE}"
        USER = "jenkins"
    }
    
    stages {
        stage('Preparación') {
            steps {
                script {
                    // Configurar permisos máximos
                    sh '''
                        umask 000
                        export UMASK=000
                        echo "🔧 Configurando permisos máximos..."
                        
                        # Crear directorio de reportes con permisos completos
                        sudo mkdir -p /var/jenkins_home/workspace/reports || true
                        sudo chmod 777 /var/jenkins_home/workspace/reports || true
                        
                        # Dar permisos completos al workspace actual
                        sudo chmod -R 777 "${WORKSPACE}" || true
                        
                        # Asegurar que el usuario jenkins tenga acceso total
                        sudo chown -R jenkins:jenkins "${WORKSPACE}" || true
                        
                        echo "🔍 Usuario Jenkins info:"
                        id
                        whoami
                        pwd
                        
                        echo "📁 Permisos del workspace:"
                        ls -la "${WORKSPACE}"
                        
                        echo "📁 Espacio disponible:"
                        df -h "${WORKSPACE}"
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
                        // Configurar permisos antes de comenzar
                        sh '''
                            umask 000
                            export UMASK=000
                            sudo chmod -R 777 "${WORKSPACE}" || true
                        '''

                        echo "🔨 Construyendo imagen Docker..."
                        sh "docker build -t ${IMAGE_NAME}:${IMAGE_TAG} ."

                        echo "🔍 Ejecutando SAST con Semgrep..."
                        sh """
                            # Asegurar permisos antes del scan
                            sudo chmod -R 777 "${WORKSPACE_DIR}" || true
                            
                            docker run --rm \
                                -v "${WORKSPACE_DIR}:/src" \
                                --workdir /src \
                                --user root \
                                returntocorp/semgrep:latest \
                                sh -c "umask 000 && semgrep scan --config auto --json --output /src/semgrep-report.json --force-color --no-git-ignore . && chmod 666 /src/semgrep-report.json" || true
                        """

                        echo "🔍 Escaneando imagen Docker con Trivy..."
                        sh """
                            sudo chmod -R 777 "${WORKSPACE_DIR}" || true
                            
                            docker run --rm \
                                -v /var/run/docker.sock:/var/run/docker.sock \
                                -v "${WORKSPACE_DIR}:/workspace" \
                                --workdir /workspace \
                                --user root \
                                aquasec/trivy:latest \
                                sh -c "umask 000 && trivy image --format json ${IMAGE_NAME}:${IMAGE_TAG} > trivy-image-report.json && chmod 666 trivy-image-report.json" || true
                        """

                        echo "🔍 Ejecutando análisis de configuración con Checkov..."
                        sh """
                            sudo chmod -R 777 "${WORKSPACE_DIR}" || true
                            
                            docker run --rm \
                                -v "${WORKSPACE_DIR}:/workspace" \
                                --workdir /workspace \
                                --user root \
                                bridgecrew/checkov:latest \
                                sh -c "umask 000 && checkov --directory . --framework dockerfile -o json --output-file-path . --soft-fail && chmod 666 *.json" || true
                        """

                        // Renombrar archivo de Checkov si tiene nombre diferente
                        sh """
                            if [ -f results_json.json ]; then
                                mv results_json.json checkov-report.json
                                chmod 666 checkov-report.json || true
                            fi
                        """

                        // Dar permisos completos a todos los archivos generados
                        sh """
                            sudo chmod 666 *.json || true
                            sudo chmod 666 *.xml || true
                            sudo chmod 666 *.html || true
                            sudo chown jenkins:jenkins *.json || true
                            sudo chown jenkins:jenkins *.xml || true
                            sudo chown jenkins:jenkins *.html || true
                        """

                        echo "✅ Verificando reportes generados..."
                        sh """
                            echo "📊 Archivos JSON generados:"
                            ls -lh *.json || echo "No hay archivos JSON aún"
                            
                            echo "📄 Verificando Semgrep específicamente:"
                            if [ -f semgrep-report.json ]; then
                                echo "✅ semgrep-report.json existe (\$(wc -c < semgrep-report.json) bytes)"
                                echo "📄 Primeras líneas del reporte Semgrep:"
                                head -3 semgrep-report.json
                            else
                                echo "❌ semgrep-report.json NO existe"
                                echo "🔍 Buscando archivos semgrep..."
                                find . -name "*semgrep*" -type f 2>/dev/null || echo "No se encontraron archivos semgrep"
                            fi
                            
                            echo "📄 Verificando Trivy:"
                            if [ -f trivy-image-report.json ]; then
                                echo "✅ trivy-image-report.json existe (\$(wc -c < trivy-image-report.json) bytes)"
                            else
                                echo "❌ trivy-image-report.json NO existe"
                            fi
                            
                            echo "📄 Verificando Checkov:"
                            if [ -f checkov-report.json ]; then
                                echo "✅ checkov-report.json existe (\$(wc -c < checkov-report.json) bytes)"
                            else
                                echo "❌ checkov-report.json NO existe"
                            fi
                            
                            echo "📄 Todos los archivos en el workspace:"
                            ls -la
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
                            # Asegurar permisos antes del scan ZAP
                            sudo chmod -R 777 "${WORKSPACE_DIR}" || true
                            
                            docker run --rm \
                                --network jenkins_jenkins-network \
                                -v "${WORKSPACE_DIR}:/workspace" \
                                --workdir /workspace \
                                --user root \
                                zaproxy/zap-stable:latest \
                                sh -c "umask 000 && zap-baseline.py \
                                    -t http://juice-shop-running:3000 \
                                    -r zap-baseline-report.html \
                                    -x zap-baseline-report.xml \
                                    -I && chmod 666 zap-baseline-report.*" || true
                        """

                        // Ajustar permisos de los reportes ZAP
                        sh """
                            sudo chmod 666 zap-baseline-report.* || true
                            
                            if [ -f zap-baseline-report.xml ]; then
                                echo "📄 Reporte ZAP XML generado exitosamente"
                                ls -lh zap-baseline-report.*
                            else
                                echo "⚠️ Reporte ZAP no encontrado"
                                ls -la . | grep zap || echo "No hay archivos ZAP"
                            fi
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

                // Dar permisos completos a todos los archivos antes del archivado
                sh """
                    echo "🔧 Configurando permisos finales..."
                    sudo chmod -R 777 "${WORKSPACE}" || true
                    sudo chmod 666 *.json || true
                    sudo chmod 666 *.xml || true
                    sudo chmod 666 *.html || true
                    sudo chown -R jenkins:jenkins "${WORKSPACE}" || true
                    
                    echo "🔍 Verificación final de archivos:"
                    echo "Archivos JSON:"
                    find . -name "*.json" -type f -exec ls -la {} \\; || echo "No hay archivos JSON"
                    echo "Archivos XML:"
                    find . -name "*.xml" -type f -exec ls -la {} \\; || echo "No hay archivos XML"  
                    echo "Archivos HTML:"
                    find . -name "*.html" -type f -exec ls -la {} \\; || echo "No hay archivos HTML"
                """

                echo "📋 RESUMEN FINAL DE REPORTES:"
                echo "============================="
                sh """
                    echo "🔍 Todos los archivos JSON generados:"
                    ls -lh *.json || echo "No hay reportes JSON"
                    
                    echo ""
                    echo "🔍 Reportes ZAP generados:"
                    ls -lh zap-baseline-report.* || echo "No hay reportes ZAP"
                    
                    echo ""
                    echo "📊 Contenido de reportes (primeras líneas):"
                    for file in *.json; do
                        if [ -f "\$file" ]; then
                            echo "=== \$file ==="
                            head -3 "\$file"
                        fi
                    done || echo "No se pueden leer reportes"
                    
                    echo ""
                    echo "📁 Permisos finales de archivos:"
                    ls -la *.json *.xml *.html 2>/dev/null || echo "No hay archivos para mostrar permisos"
                """

                // Archivar todos los reportes de seguridad
                archiveArtifacts artifacts: '*.json, zap-baseline-report.*, *.xml, *.html', fingerprint: true, allowEmptyArchive: true
                
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