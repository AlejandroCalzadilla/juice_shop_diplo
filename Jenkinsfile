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
        
        stage('SAST - Semgrep') {
            steps {
                script {
                    try {
                        sh """
                            docker run --rm \
                                -v "\${PWD}:/src" \
                                --workdir /src \
                                returntocorp/semgrep:latest \
                                semgrep scan --config auto --verbose
                        """
                    } catch (Exception e) {
                        echo "⚠️ Semgrep falló, continuando: ${e.getMessage()}"
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
                                -v \${PWD}:/workspace \
                                --workdir /workspace \
                                aquasec/trivy:latest \
                                fs . --timeout 5m
                        """
                    } catch (Exception e) {
                        echo "⚠️ Trivy SCA falló, continuando: ${e.getMessage()}"
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
                        
                        // Escanear imagen
                        sh """
                            docker run --rm \
                                -v /var/run/docker.sock:/var/run/docker.sock \
                                aquasec/trivy:latest \
                                image --timeout 10m \
                                ${IMAGE_NAME}:${IMAGE_TAG}
                        """
                        
                    } catch (Exception e) {
                        echo "⚠️ Build o Image Scan falló: ${e.getMessage()}"
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
                        
                        // Ejecutar ZAP scan
                        sh """
                            docker run --rm \
                                --network jenkins_jenkins-network \
                                zaproxy/zap-stable:latest \
                                zap-baseline.py \
                                    -t http://juice-shop-running:3000 \
                                    -I || true
                        """
                        
                    } catch (Exception e) {
                        echo "⚠️ DAST falló: ${e.getMessage()}"
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
                                --directory . || true
                        """
                    } catch (Exception e) {
                        echo "⚠️ Checkov falló: ${e.getMessage()}"
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