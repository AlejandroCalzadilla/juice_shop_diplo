pipeline {
    agent any
    
    environment {
        REPORTS_DIR = "${WORKSPACE}/reports"
        IMAGE_NAME = "juice-shop"
        IMAGE_TAG = "latest"
    }
    
    stages {
        stage('Preparación') {
            steps {
                script {
                    sh "mkdir -p ${REPORTS_DIR}"
                    sh "docker stop juice-shop-running || true"
                    sh "docker rm juice-shop-running || true"
                    sh "docker rmi ${IMAGE_NAME}:${IMAGE_TAG} || true"
                }
            }
        }
        
        stage('Checkout') {
            steps {
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
                        sh "docker build -t ${IMAGE_NAME}:${IMAGE_TAG} ."
                        sh """
                            docker run --rm \
                                -v /var/run/docker.sock:/var/run/docker.sock \
                                aquasec/trivy:latest \
                                image --format json \
                                --output /tmp/image-report.json \
                                ${IMAGE_NAME}:${IMAGE_TAG}
                        """
                        sh "docker run --rm -v \${PWD}:/host -v /tmp:/tmp alpine cp /tmp/image-report.json /host/reports/image-report.json || echo '{}' > ${REPORTS_DIR}/image-report.json"
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
                        sh """
                            docker run -d --rm \
                                --name juice-shop-running \
                                -p 3000:3000 \
                                ${IMAGE_NAME}:${IMAGE_TAG}
                        """
                        sh "sleep 45"
                        sh "curl -f http://localhost:3000 || (echo 'Aplicación no responde' && exit 1)"
                        sh """
                            docker run --rm \
                                --network host \
                                -v \${PWD}/reports:/zap/wrk/:rw \
                                owasp/zap2docker-stable \
                                zap-baseline.py \
                                -t http://127.0.0.1:3000 \
                                -J dast-report.json || true
                        """
                    } catch (Exception e) {
                        echo "⚠️ DAST falló: ${e.getMessage()}"
                        sh "echo '{}' > ${REPORTS_DIR}/dast-report.json"
                    } finally {
                        sh "docker stop juice-shop-running || true"
                    }
                }
            }
        }
        
        stage('PaC - Checkov') {
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
            sh "docker stop juice-shop-running || true"
            sh "docker rm juice-shop-running || true"
            archiveArtifacts artifacts: 'reports/*.json', fingerprint: true, allowEmptyArchive: true
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