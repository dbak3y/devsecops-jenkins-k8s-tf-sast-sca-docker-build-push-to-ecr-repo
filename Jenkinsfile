pipeline {
  agent any

  environment {
    // Tool names (change if different in your Jenkins)
    MAVEN_TOOL = 'Maven_3_9_6'

    // Project / repo variables
    IMAGE_NAME = "asg"                                      // docker image short name
    ECR_REGISTRY = "268428820004.dkr.ecr.us-west-2.amazonaws.com"
    ECR_IMAGE = "${ECR_REGISTRY}/${IMAGE_NAME}"

    // Kubernetes namespace & service
    K8S_NAMESPACE = "devsecops"
    K8S_SERVICE = "asgbuggy"

    // Timeouts
    DEPLOY_WAIT_SECONDS = 180
  }

  tools {
    maven "${env.MAVEN_TOOL}"
  }

  options {
    ansiColor('xterm')
    timestamps()
    // keep builds for >= 30 runs
    buildDiscarder(logRotator(numToKeepStr: '30'))
  }

  stages {

    stage('Checkout') {
      steps {
        checkout scm
        script {
          echo "Checked out ${env.GIT_COMMIT ?: 'no commit info'}"
        }
      }
    }

    stage('Build & Unit Tests & Sonar') {
      environment {
        // Sonar token stored in Jenkins credentials as secret text (create this if missing)
        // Replace 'SONAR_TOKEN_ID' with your credential ID
      }
      steps {
        withCredentials([string(credentialsId: 'SONAR_TOKEN_ID', variable: 'SONAR_TOKEN')]) {
          sh '''
            set -eux
            mvn -B clean verify
            mvn -B sonar:sonar \
              -Dsonar.projectKey=dbak3ybugywebapp \
              -Dsonar.organization=dbak3ybugywebapp \
              -Dsonar.host.url=https://sonarcloud.io \
              -Dsonar.login=$SONAR_TOKEN
          '''
        }
      }
      post {
        always {
          junit '**/target/surefire-reports/*.xml'
        }
      }
    }

    stage('SCA - Snyk (Open Source + IaC + Container)') {
      steps {
        withCredentials([string(credentialsId: 'SNYK_TOKEN', variable: 'SNYK_TOKEN')]) {
          // Use official snyk docker image to avoid needing snyk on the agent
          script {
            // Test source code and produce JSON output
            docker.image('snyk/snyk:1.1038.0').inside("--entrypoint=''") {
              sh '''
                set -eux
                echo "$SNYK_TOKEN" | snyk auth --stdin
                snyk test --all-projects || true    # don't fail pipeline here; control via policy if you want
                snyk monitor --all-projects
              '''
            }
          }
        }
      }
    }

    stage('Build Docker Image') {
      steps {
        script {
          docker.withRegistry('', 'dockerlogin') {
            // Build local docker image with tag latest (we'll tag & push to ECR next)
            app = docker.build("${IMAGE_NAME}:${env.BUILD_NUMBER}")
          }
        }
      }
    }

    stage('Authenticate ECR & Push') {
      steps {
        script {
          // Use AWS ECR credentials stored as credential helper 'ecr:us-west-2:aws-credentials'
          docker.withRegistry("https://${ECR_REGISTRY}", 'ecr:us-west-2:aws-credentials') {
            // Tag & push
            sh "docker tag ${IMAGE_NAME}:${env.BUILD_NUMBER} ${ECR_IMAGE}:${env.BUILD_NUMBER}"
            sh "docker tag ${IMAGE_NAME}:${env.BUILD_NUMBER} ${ECR_IMAGE}:latest"
            sh "docker push ${ECR_IMAGE}:${env.BUILD_NUMBER}"
            sh "docker push ${ECR_IMAGE}:latest"
          }
        }
      }
    }

    stage('Kubernetes Deployment') {
      steps {
        // withKubeConfig injects KUBECONFIG for kubectl commands inside the block
        withKubeConfig([credentialsId: 'kubelogin']) {
          sh '''
            set -eux
            # namespace create if not exists
            kubectl get namespace ${K8S_NAMESPACE} || kubectl create namespace ${K8S_NAMESPACE}

            # Replace image in deployment manifest (supports docker image placeholders)
            # If your deployment.yaml references image directly, you can use kubectl set image instead.
            kubectl apply -f deployment.yaml --namespace=${K8S_NAMESPACE}

            # If your YAML doesn't parameterize image, update the deployment image explicitly:
            kubectl set image deployment/asgbuggy asgbuggy=${ECR_IMAGE}:latest -n ${K8S_NAMESPACE} || true
          '''
        }
      }
    }

    stage('Wait for App') {
      steps {
        withKubeConfig([credentialsId: 'kubelogin']) {
          script {
            // wait for service to get an external LB hostname
            timeout(time: 6, unit: 'MINUTES') {
              echo "Waiting up to ${DEPLOY_WAIT_SECONDS}s for service ${K8S_SERVICE} to have an external LB..."
              waitUntil {
                def host = sh(
                  script: "kubectl get svc ${K8S_SERVICE} -n ${K8S_NAMESPACE} -o jsonpath='{.status.loadBalancer.ingress[0].hostname}' || true",
                  returnStdout: true
                ).trim()
                if (host) {
                  env.TARGET_HOST = host
                  echo "Found hostname: ${host}"
                  return true
                } else {
                  echo "LB hostname not yet available; sleeping 10s..."
                  sleep 10
                  return false
                }
              }
            }
          }
        }
      }
    }

    stage('DAST - OWASP ZAP (Quick Scan)') {
      steps {
        script {
          // Ensure TARGET_HOST is present (set by previous stage)
          if (!env.TARGET_HOST) {
            error "TARGET_HOST is not set: cannot run DAST"
          }

          // Run zap inside its official docker container. We will mount workspace so report is accessible.
          docker.image('owasp/zap2docker-stable').inside("-u root -v ${env.WORKSPACE}:${env.WORKSPACE}") {
            // Use bash block to run zap.sh
            sh """
              set -eux
              export TARGET_URL="http://${env.TARGET_HOST}"
              echo "Running ZAP against $TARGET_URL"
              # quick scan - adjust flags as needed; increase timeouts for bigger apps
              zap.sh -cmd -quickurl $TARGET_URL -quickprogress -quickout ${WORKSPACE}/zap_report.html
            """
          }
        }
      }
      post {
        always {
          archiveArtifacts artifacts: 'zap_report.html', allowEmptyArchive: true
        }
      }
    }

    stage('Post-checks & Cleanup') {
      steps {
        // Optional smoke test: hit root and confirm 200
        withKubeConfig([credentialsId: 'kubelogin']) {
          sh '''
            set -eux
            if [ -n "${TARGET_HOST}" ]; then
              curl -sS --fail "http://${TARGET_HOST}" || echo "Warning: app root not returning 200"
            else
              echo "TARGET_HOST empty - skipping smoke curl"
            fi
          '''
        }
      }
    }
  } // stages

  post {
    success {
      echo "Pipeline completed successfully."
    }
    failure {
      echo "Pipeline failed - check logs and artifacts (ZAP report)."
    }
    always {
      // optional: keep a small summary file
      sh 'echo "Pipeline finished at $(date) (result: ${BUILD_STATUS})" > pipeline_finish.txt || true'
      archiveArtifacts artifacts: 'pipeline_finish.txt', allowEmptyArchive: true
    }
  }
}
