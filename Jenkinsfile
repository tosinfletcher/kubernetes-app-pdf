pipeline {
    agent any

    parameters {
        string(name: 'BRANCH_NAME', defaultValue: 'main', description: 'Branch to deploy')
        string(name: 'CP_IP_ADDRESS', defaultValue: '192.168.0.60', description: 'Hostname of the Control Plane Kubernetes Cluster')
        string(name: 'CP_SSH_USER', defaultValue: 'tfletcher', description: 'SSH username of the Control Plane Kubernetes Cluster')
        string(name: 'CP_PRIVATE_KEY_ID', defaultValue: 'cp-private-key', description: 'Private Key ID of the Control Plane Kubernetes Cluster')
    }

    environment {
        JENKINS_JOB_WEBHOOK_URL = credentials('jenkins-job-webhook-url')
        PROJECT_NAME = "kubernetes-app-pdf"
        APPS_DIR = 'apps'
    }

    stages {
        stage("Checkout") {
            steps {
                deleteDir()
                checkout scm
            }
        }

        stage("Deploy PDF Service") {
            steps {
                sshagent([params.CP_PRIVATE_KEY_ID]) {
                    sh """
                    # Ensure directory apps directory exist
                    ssh -o StrictHostKeyChecking=no ${params.CP_SSH_USER}@${params.CP_IP_ADDRESS} \\
                        "mkdir -p /home/${params.CP_SSH_USER}/${env.APPS_DIR}/${env.PROJECT_NAME}"

                    # Remove contents of applications directory but keep the directory itself
                    ssh -o StrictHostKeyChecking=no ${params.CP_SSH_USER}@${params.CP_IP_ADDRESS} \\
                        "find /home/${params.CP_SSH_USER}/${env.APPS_DIR}/${env.PROJECT_NAME} -mindepth 1 -delete"

                    # Securely copy the application directory to the target directory on CP Kubernetes Cluster
                    scp -o StrictHostKeyChecking=no -r . ${params.CP_SSH_USER}@${params.CP_IP_ADDRESS}:/home/${params.CP_SSH_USER}/${env.APPS_DIR}/${env.PROJECT_NAME}/

                    # Verify files were copied
                    ssh -o StrictHostKeyChecking=no ${params.CP_SSH_USER}@${params.CP_IP_ADDRESS} \\
                        "cd /home/${params.CP_SSH_USER}/${env.APPS_DIR}/${env.PROJECT_NAME} && ls -la"
                    
                    ssh -o StrictHostKeyChecking=no ${params.CP_SSH_USER}@${params.CP_IP_ADDRESS} <<EOF

                    # Deploy application
                    cd /home/${params.CP_SSH_USER}/${env.APPS_DIR}/${env.PROJECT_NAME}/

                    microk8s helm3 upgrade --install pdf . --kubeconfig /home/${params.CP_SSH_USER}/.kube/config

EOF
                    """
                }
            }
        }

        stage('Post-Deploy Check') {
            steps {
                sshagent([params.CP_PRIVATE_KEY_ID]) {
                    sh """
                        echo "[+] Verifying running pod for project ${env.PROJECT_NAME}..."

                        ssh -o StrictHostKeyChecking=no ${params.CP_SSH_USER}@${params.CP_IP_ADDRESS} <<EOF
                        cd /home/${params.CP_SSH_USER}/${env.APPS_DIR}/${env.PROJECT_NAME}/

                        microk8s kubectl get pods
EOF
                    """
                }
            }
        }
    }

    post {
        always {
            echo '[+] Cleaning up workspace and sensitive files'
            cleanWs()
        }
        success {
            echo '[✓] Deployment successful!'
        }
        failure {
            echo '[x] Deployment failed.'
            // Send failure notification to pumble
            sh """
                curl -X POST -H 'Content-type: application/json' \\
                --data '{
                    "text": "🚨 Build Failed! Job: `${env.JOB_NAME}` Build: `${env.BUILD_NUMBER}`. Check logs: ${env.BUILD_URL}"
                }' \\
                ${JENKINS_JOB_WEBHOOK_URL}
            """
        }
    }
}