pipeline {
    agent any

    environment {
        REGISTRY      = "registry.local:5443"
        IMAGE_NAME    = "juiceshop"
        IMAGE_TAG     = "${BUILD_NUMBER}"
    }

    stages {

        // =============================
        // Docker Build & Push
        // =============================
        stage('Docker Build & Push') {
            steps {
                script {
                    withCredentials([
                        usernamePassword(
                            credentialsId: 'registry-credentials',
                            usernameVariable: 'REG_USER',
                            passwordVariable: 'REG_PASS'
                        )
                    ]) {
                        sh """
                            echo \$REG_PASS | docker login ${REGISTRY} \
                                -u \$REG_USER --password-stdin

                            docker build -t ${REGISTRY}/${IMAGE_NAME}:${IMAGE_TAG} .
                            docker tag ${REGISTRY}/${IMAGE_NAME}:${IMAGE_TAG} \
                                       ${REGISTRY}/${IMAGE_NAME}:latest

                            docker push ${REGISTRY}/${IMAGE_NAME}:${IMAGE_TAG}
                            docker push ${REGISTRY}/${IMAGE_NAME}:latest

                            docker logout ${REGISTRY}
                        """
                    }
                }
            }
        }

        // =============================
        // Trivy Scan + Send to n8n
        // =============================
        stage('Trivy Scan + Trigger n8n') {
            steps {
                script {
                    withCredentials([
                        usernamePassword(
                            credentialsId: 'registry-credentials',
                            usernameVariable: 'REG_USER',
                            passwordVariable: 'REG_PASS'
                        )
                    ]) {
                        sh """
                            echo \$REG_PASS | docker login ${REGISTRY} \
                                -u \$REG_USER --password-stdin

                            docker pull ${REGISTRY}/${IMAGE_NAME}:${IMAGE_TAG}

                            trivy image \
                                --scanners vuln \
                                --severity HIGH,CRITICAL \
                                --format json \
                                --output /tmp/trivy-latest.json \
                                ${REGISTRY}/${IMAGE_NAME}:${IMAGE_TAG}

                            echo "Trivy scan terminé"

                            curl -s -X POST \
                                http://192.168.56.21:5678/webhook-test/trivy-scan \
                                -H "Content-Type: application/json" \
                                --data-binary @/tmp/trivy-latest.json

                            echo "n8n notifié"
                        """
                    }
                }
            }
        }
    }
}
