pipeline {
    agent any

    tools { 
        maven 'Maven_3_9_6'
    }

    environment {
        SONAR_PROJECT_KEY = 'dbak3ybugywebapp'
        SONAR_ORG         = 'dbak3ybugywebapp'
        SONAR_HOST_URL    = 'https://sonarcloud.io'

        DOCKER_IMAGE = 'asg'
        ECR_REGISTRY = '268428820004.dkr.ecr.us-west-2.amazonaws.com'
        K8S_NAMESPACE = 'devsecops'
    }

    stages {

        stage('Compile & Sonar Analysis') {
            steps {
                // In a real setup, use withCredentials + SONAR_TOKEN from Jenkins
                sh """
                    mvn clean verify sonar:sonar \
                      -Dsonar.projectKey=${SONAR_PROJECT_KEY} \
                      -Dsonar.organization=${SONAR_ORG} \
                      -Dsonar.host.url=${SONAR_HOST_URL} \
                      -Dsonar.token=5169cafefbe8232c4f60dbbbbaf3995de3e750fb
                """
            }
        }

        stage('SCA Analysis with Snyk') {
            steps {
                withCredentials([string(credentialsId: 'SNYK_TOKEN', variable: 'SNYK_TOKEN')]) {
                    sh """
                        mvn snyk:test \
                          -Dsnyk.token=${SNYK_TOKEN} \
                          -fn
                    """
                }
            }
        }

        stage('Build Docker Image') {
            steps {
                withDockerRegistry([credentialsId: 'dockerlogin', url: '']) {
                    script {
                        app = docker.build("${DOCKER_IMAGE}")
                    }
                }
            }
        }

        stage('Push Docker Image to ECR') {
            steps {
                script {
                    docker.withRegistry("https://${ECR_REGISTRY}", 'ecr:us-west-2:aws-credentials') {
                        app.push("latest")
                    }
                }
            }
        }

        stage('Kubernetes Deployment of ASG Buggy Web Application') {
            steps {
                withKubeConfig([credentialsId: 'kubelogin']) {
                    // Brutal reset of namespace – fine for lab, not for prod
                    sh "kubectl delete all --all -n ${K8S_NAMESPACE} || true"
                    sh "kubectl apply -f deployment.yaml --namespace=${K8S_NAMESPACE}"
                }
            }
        }

        stage('Wait for Deployment') {
            steps {
                sh 'echo "Waiting 180 seconds for app to be ready"; sleep 180'
            }
        }

        stage('Run DAST Using ZAP') {
            steps {
                withKubeConfig([credentialsId: 'kubelogin']) {
                    script {
                        // Get ELB hostname from the service
                        def serviceUrl = sh(
                            script: "kubectl get service/asgbuggy -n ${K8S_NAMESPACE} -o json | jq -r '.status.loadBalancer.ingress[0].hostname'",
                            returnStdout: true
                        ).trim()

                        echo "ZAP scanning URL: http://${serviceUrl}"

                        // Ensure a writable folder for ZAP inside the workspace
                        sh """
                            mkdir -p "${WORKSPACE}/zap-output"
                            chmod -R 777 "${WORKSPACE}/zap-output"
                        """

                        def maxRetries = 3
                        def success = false

                        for (int i = 1; i <= maxRetries; i++) {
                            try {
                                sh """
                                    docker run --rm \
                                      -v ${WORKSPACE}/zap-output:/zap/wrk:rw \
                                      ghcr.io/zaproxy/zaproxy:latest \
                                      zap.sh -cmd \
                                        -quickurl http://${serviceUrl} \
                                        -quickprogress \
                                        -quickout /zap/wrk/zap_report.html \
                                        -exclude '^.*/oom.*' \
                                        -exclude '^.*/memory.*' \
                                        -exclude '^.*/thread.*' \
                                        -exclude '^.*/slow.*' \
                                        -exclude '^.*/illegal.*' \
                                        -exclude '^.*/massive.*' \
                                        -exclude '^.*/exception.*'
                                """

                                if (!fileExists("${WORKSPACE}/zap-output/zap_report.html")) {
                                    error "ZAP report not found after scan!"
                                }

                                success = true
                                break

                            } catch (Exception e) {
                                echo "ZAP Docker run failed, retry ${i}/${maxRetries}..."
                                if (i == maxRetries) {
                                    error "ZAP scan failed after ${maxRetries} attempts."
                                }
                                sleep 30
                            }
                        }

                        archiveArtifacts artifacts: 'zap-output/zap_report.html'
                    }
                }
            }
        }
    }

    post {
        always {
            echo "Pipeline completed (success or fail)."
        }
        success {
            echo "✅ Pipeline finished successfully."
        }
        failure {
            echo "❌ Pipeline failed – check the stage logs above."
        }
    }
}
