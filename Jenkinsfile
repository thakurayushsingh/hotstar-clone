pipeline {
    agent any

    tools {
        jdk 'jdk-17'
        nodejs 'Node16'
    }

    parameters {
        choice(name: 'ENVIRONMENT', choices: ['dev', 'staging', 'prod'], description: 'Target deployment environment')
        choice(name: 'DEPLOY_METHOD', choices: ['Kubernetes', 'Ansible', 'Docker-Container'], description: 'Deployment orchestrator method')
        string(name: 'IMAGE_TAG', defaultValue: 'latest', description: 'Docker image tag (e.g., latest or build number)')
        booleanParam(name: 'ENABLE_SECURITY_SCANS', defaultValue: true, description: 'Run OWASP Dependency-Check & Trivy vulnerability scans')
        booleanParam(name: 'RUN_ANSIBLE_PROVISION', defaultValue: false, description: 'Run Ansible playbook to provision target host')
    }

    environment {
        SCANNER_HOME = tool 'sonar-scanner'
        DOCKER_IMAGE = "thakurayushsingh/hotstar:${params.IMAGE_TAG}"
        REPO_URL     = 'https://github.com/thakurayushsingh/hotstar-clone.git'
    }

    stages {
        stage('Clean Workspace') {
            steps {
                cleanWs()
            }
        }

        stage('Git Checkout') {
            steps {
                git branch: 'main', url: "${env.REPO_URL}"
            }
        }

        stage('SonarQube Static Analysis') {
            steps {
                withSonarQubeEnv('sonar-server') {
                    sh """
                        $SCANNER_HOME/bin/sonar-scanner \
                        -Dsonar.projectName=Disney-Hotstar-Clone \
                        -Dsonar.projectKey=hotstar-clone
                    """
                }
            }
        }

        stage('Quality Gate Check') {
            steps {
                script {
                    waitForQualityGate abortPipeline: false, credentialsId: 'Sonar-token'
                }
            }
        }

        stage('Install Dependencies') {
            steps {
                sh 'npm install'
            }
        }

        stage('OWASP Dependency-Check') {
            when {
                expression { return params.ENABLE_SECURITY_SCANS }
            }
            steps {
                dependencyCheck additionalArguments: '--scan ./ --disableYarnAudit --disableNodeAudit', odcInstallation: 'DP'
                dependencyCheckPublisher pattern: '**/dependency-check-report.xml'
            }
        }

        stage('Trivy Filesystem Scan') {
            when {
                expression { return params.ENABLE_SECURITY_SCANS }
            }
            steps {
                sh 'trivy fs --severity HIGH,CRITICAL . > trivyfs.txt || true'
            }
        }

        stage('Docker Build & Push') {
            steps {
                script {
                    withDockerRegistry(credentialsId: 'docker', toolName: 'docker') {
                        sh "docker build -t ${env.DOCKER_IMAGE} ."
                        sh "docker push ${env.DOCKER_IMAGE}"
                    }
                }
            }
        }

        stage('Trivy Container Image Scan') {
            when {
                expression { return params.ENABLE_SECURITY_SCANS }
            }
            steps {
                sh "trivy image --severity HIGH,CRITICAL ${env.DOCKER_IMAGE} > trivyimage.txt || true"
            }
        }

        stage('Ansible Host Configuration') {
            when {
                expression { return params.RUN_ANSIBLE_PROVISION }
            }
            steps {
                sh '''
                    if [ -f "ansible/inventory.ini" ]; then
                        ansible-playbook -i ansible/inventory.ini ansible/playbooks/setup-docker.yml || true
                    fi
                '''
            }
        }

        stage('Deploy Application') {
            steps {
                script {
                    switch(params.DEPLOY_METHOD) {
                        case 'Kubernetes':
                            echo "Deploying to Kubernetes in ${params.ENVIRONMENT} environment..."
                            sh 'kubectl apply -f K8S/manifest.yml'
                            sh 'kubectl rollout status deployment/hotstar-deployment --timeout=120s || true'
                            break

                        case 'Ansible':
                            echo "Deploying via Ansible to ${params.ENVIRONMENT} nodes..."
                            sh "ansible-playbook -i ansible/inventory.ini ansible/playbooks/deploy-app.yml --extra-vars 'image_tag=${params.IMAGE_TAG}' || true"
                            break

                        case 'Docker-Container':
                            echo "Deploying to local Docker container for ${params.ENVIRONMENT}..."
                            sh "docker rm -f hotstar-${params.ENVIRONMENT} || true"
                            sh "docker run -d --name hotstar-${params.ENVIRONMENT} -p 3000:3000 ${env.DOCKER_IMAGE}"
                            break
                    }
                }
            }
        }
    }

    post {
        always {
            script {
                def buildStatus = currentBuild.currentResult
                echo "Pipeline finished with status: ${buildStatus} for Environment: ${params.ENVIRONMENT} using: ${params.DEPLOY_METHOD}"

                // Optional Slack notification (if Slack plugin is installed and configured)
                try {
                    slackSend(
                        channel: '#devops-deployments',
                        color: buildStatus == 'SUCCESS' ? '#00FF00' : '#FF0000',
                        message: "Disney+ Hotstar Pipeline [${params.ENVIRONMENT}]: ${buildStatus} - Job ${env.JOB_NAME} #${env.BUILD_NUMBER} (${env.BUILD_URL})"
                    )
                } catch (Exception e) {
                    echo "Slack notification skipped or plugin not configured."
                }
            }
        }
        success {
            echo "Deployment to ${params.ENVIRONMENT} succeeded!"
        }
        failure {
            echo "Deployment failed! Check build console logs."
        }
    }
}
