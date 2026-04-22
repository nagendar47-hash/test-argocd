pipeline {
    agent any

    environment {
        GIT_REPO      = "https://github.com/your-org/k8s-manifests.git"
        ARGOCD_SERVER = "192.168.64.6:32443"      // ← your node IP : argocd nodeport
        APP_NAME      = "my-app"
    }

    stages {

        stage('Checkout Manifests Repo') {
            steps {
                echo "---- Cloning K8s Manifests Repo ----"
                git branch: 'main',
                    url: "${GIT_REPO}",
                    credentialsId: 'GIT_CREDENTIALS'
            }
        }

        stage('Validate Manifests') {
            steps {
                echo "---- Validating K8s YAML files ----"
                sh """
                    kubectl apply --dry-run=client \
                        -f apps/my-app/deployment.yaml

                    kubectl apply --dry-run=client \
                        -f apps/my-app/service.yaml

                    echo "✅ All manifests are valid!"
                """
            }
        }

        stage('Sync via ArgoCD') {
            steps {
                echo "---- Triggering ArgoCD Sync ----"
                withCredentials([string(
                    credentialsId: 'ARGOCD_TOKEN',
                    variable: 'ARGOCD_AUTH_TOKEN'
                )]) {
                    sh """
                        argocd app sync ${APP_NAME} \
                            --server ${ARGOCD_SERVER} \
                            --auth-token \$ARGOCD_AUTH_TOKEN \
                            --insecure

                        echo "---- Waiting for Sync to Complete ----"
                        argocd app wait ${APP_NAME} \
                            --server ${ARGOCD_SERVER} \
                            --auth-token \$ARGOCD_AUTH_TOKEN \
                            --insecure \
                            --health \
                            --timeout 120
                    """
                }
            }
        }

        stage('Verify Deployment') {
            steps {
                echo "---- Verifying Pods are Running ----"
                sh """
                    kubectl get pods -n default -l app=my-app
                    kubectl rollout status deployment/my-app -n default
                    echo "✅ Deployment verified successfully!"
                """
            }
        }
    }

    post {
        success {
            echo "✅ Pipeline SUCCESS — my-app is running on cluster!"
        }
        failure {
            echo "❌ Pipeline FAILED — Check ArgoCD UI or kubectl logs"
        }
        always {
            echo "---- App Status ----"
            sh """
                argocd app get ${APP_NAME} \
                    --server ${ARGOCD_SERVER} \
                    --auth-token \$ARGOCD_AUTH_TOKEN \
                    --insecure || true
            """
        }
    }
}
