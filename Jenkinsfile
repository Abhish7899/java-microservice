pipeline {
    agent {
        label 'ephemeral-ec2'
    }

    options {
        timestamps()
        disableConcurrentBuilds()
        skipDefaultCheckout(true)
        timeout(time: 30, unit: 'MINUTES')
        buildDiscarder(logRotator(numToKeepStr: '20'))
    }

    parameters {
        string(
            name: 'BASE_VERSION',
            defaultValue: '1.0',
            description: 'Major.Minor prefix (auto-appends Jenkins BUILD_NUMBER -> e.g. 1.0.17)'
        )
    }

    environment {
        AWS_REGION     = 'ap-southeast-2'
        AWS_ACCOUNT_ID = '297681905096'
        ECR_REGISTRY   = "${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com"
        ECR_REPOSITORY = 'java-microservice'
        S3_BUCKET      = 'java-microservice-artifacts-297681905096'
        GIT_REPO_URL   = 'https://github.com/Abhish7899/java-microservice.git'
        GIT_BRANCH     = 'main'
        NOTIFY_EMAIL   = 'dhurwadeabhishek1@gmail.com'
    }

    stages {

        stage('Checkout') {
            steps {
                git branch: "${env.GIT_BRANCH}", url: "${env.GIT_REPO_URL}"

                script {
                    env.GIT_SHORT_SHA = sh(script: 'git rev-parse --short HEAD', returnStdout: true).trim()
                    env.GIT_FULL_SHA  = sh(script: 'git rev-parse HEAD', returnStdout: true).trim()
                }

                sh '''#!/bin/bash
                    set -euo pipefail
                    echo "Checked out ${GIT_REPO_URL} (${GIT_BRANCH}) @ ${GIT_SHORT_SHA}"
                    git log -1 --oneline
                '''
            }
        }

        stage('Verify Agent Environment') {
            steps {
                sh '''#!/bin/bash
                    set -euo pipefail
                    java -version
                    mvn -version
                    docker --version
                    docker info > /dev/null
                    aws --version
                    aws sts get-caller-identity
                '''
            }
        }

        stage('Validate & Set Version') {
            steps {
                script {
                    def baseVer = (params.BASE_VERSION ?: '1.0').trim()

                    if (!(baseVer ==~ /^[0-9]+\.[0-9]+$/)) {
                        error("Invalid BASE_VERSION '${baseVer}'! Use Major.Minor format like 1.0 or 1.1")
                    }

                    env.APP_VERSION     = "${baseVer}.${env.BUILD_NUMBER}"
                    env.JAR_FILENAME    = "java-microservice-${env.APP_VERSION}.jar"
                    env.S3_ARTIFACT_URI = "s3://${env.S3_BUCKET}/java-microservice/${env.APP_VERSION}/${env.JAR_FILENAME}"
                    env.ECR_IMAGE_TAG   = "${env.APP_VERSION}"
                    env.ECR_SHA_TAG     = "${env.APP_VERSION}-${env.GIT_SHORT_SHA}"

                    echo "Synced Release Version : ${env.APP_VERSION}"
                    echo "S3 Artifact URI        : ${env.S3_ARTIFACT_URI}"
                    echo "ECR Image Tags         : ${env.ECR_IMAGE_TAG}, ${env.ECR_SHA_TAG}"
                }

                sh '''#!/bin/bash
                    set -euo pipefail
                    mvn -B versions:set \
                        -DnewVersion="${APP_VERSION}" \
                        -DgenerateBackupPoms=false
                '''
            }
        }

        stage('Unit Tests') {
            steps {
                sh '''#!/bin/bash
                    set -euo pipefail
                    mvn -B clean test
                '''
            }
            post {
                always {
                    junit allowEmptyResults: true, testResults: '**/target/surefire-reports/*.xml'
                }
            }
        }

        stage('Package JAR') {
            steps {
                sh '''#!/bin/bash
                    set -euo pipefail
                    mvn -B package -DskipTests

                    test -f "target/${JAR_FILENAME}"
                    ls -lh "target/${JAR_FILENAME}"
                '''
            }
        }

        stage('Upload Versioned JAR to S3') {
            steps {
                sh '''#!/bin/bash
                    set -euo pipefail

                    aws s3api head-bucket \
                        --bucket "${S3_BUCKET}" \
                        --expected-bucket-owner "${AWS_ACCOUNT_ID}" \
                        --region "${AWS_REGION}"

                    cat > build-info.txt <<EOF
App Version: ${APP_VERSION}
Build Number: ${BUILD_NUMBER}
Git Commit: ${GIT_FULL_SHA}
ECR Image: ${ECR_REGISTRY}/${ECR_REPOSITORY}:${ECR_IMAGE_TAG}
ECR SHA Image: ${ECR_REGISTRY}/${ECR_REPOSITORY}:${ECR_SHA_TAG}
S3 Artifact: ${S3_ARTIFACT_URI}
Build Date: $(date -u +"%Y-%m-%dT%H:%M:%SZ")
EOF

                    aws s3 cp "target/${JAR_FILENAME}" "${S3_ARTIFACT_URI}" \
                        --region "${AWS_REGION}" \
                        --sse AES256 \
                        --only-show-errors

                    aws s3 cp build-info.txt "s3://${S3_BUCKET}/java-microservice/${APP_VERSION}/build-info.txt" \
                        --region "${AWS_REGION}" \
                        --sse AES256 \
                        --only-show-errors

                    aws s3 ls "s3://${S3_BUCKET}/java-microservice/${APP_VERSION}/" --region "${AWS_REGION}"
                '''

                archiveArtifacts artifacts: 'build-info.txt', fingerprint: true
            }
        }

        stage('Build & Push Docker Image to ECR') {
            steps {
                sh '''#!/bin/bash
                    set -euo pipefail

                    aws ecr get-login-password --region "${AWS_REGION}" \
                    | docker login --username AWS --password-stdin "${ECR_REGISTRY}"

                    docker build \
                        --build-arg APP_VERSION="${APP_VERSION}" \
                        -t "${ECR_REGISTRY}/${ECR_REPOSITORY}:${ECR_IMAGE_TAG}" \
                        -t "${ECR_REGISTRY}/${ECR_REPOSITORY}:${ECR_SHA_TAG}" \
                        .

                    docker push "${ECR_REGISTRY}/${ECR_REPOSITORY}:${ECR_IMAGE_TAG}"
                    docker push "${ECR_REGISTRY}/${ECR_REPOSITORY}:${ECR_SHA_TAG}"

                    aws ecr describe-images \
                        --repository-name "${ECR_REPOSITORY}" \
                        --region "${AWS_REGION}" \
                        --image-ids imageTag="${ECR_IMAGE_TAG}" > /dev/null
                '''
            }
        }
    }

    post {
        success {
            withCredentials([string(credentialsId: 'resend-api-key', variable: 'RESEND_API_KEY')]) {
                sh '''#!/bin/bash
                    set -euo pipefail
                    cat > email-payload.json <<EOF
{
  "from": "Jenkins CI <onboarding@resend.dev>",
  "to": ["${NOTIFY_EMAIL}"],
  "subject": "SUCCESS: ${JOB_NAME} #${BUILD_NUMBER} (v${APP_VERSION})",
  "text": "CI Pipeline Succeeded\\n\\nJob: ${JOB_NAME}\\nBuild: #${BUILD_NUMBER}\\nVersion: ${APP_VERSION}\\nGit Commit: ${GIT_SHORT_SHA}\\nS3 Artifact: ${S3_ARTIFACT_URI}\\nECR Image: ${ECR_REGISTRY}/${ECR_REPOSITORY}:${ECR_IMAGE_TAG}\\nConsole Logs: ${BUILD_URL}console"
}
EOF
                    curl -fsS -X POST "https://api.resend.com/emails" \
                        -H "Authorization: Bearer ${RESEND_API_KEY}" \
                        -H "Content-Type: application/json" \
                        --data-binary @email-payload.json || echo "WARNING: Resend notification failed"
                '''
            }
        }

        failure {
            withCredentials([string(credentialsId: 'resend-api-key', variable: 'RESEND_API_KEY')]) {
                sh '''#!/bin/bash
                    set -euo pipefail
                    TARGET_VER="${APP_VERSION:-${BASE_VERSION:-unknown}}"
                    cat > email-payload.json <<EOF
{
  "from": "Jenkins CI <onboarding@resend.dev>",
  "to": ["${NOTIFY_EMAIL}"],
  "subject": "FAILED: ${JOB_NAME} #${BUILD_NUMBER} (v${TARGET_VER})",
  "text": "CI Pipeline Failed\\n\\nJob: ${JOB_NAME}\\nBuild: #${BUILD_NUMBER}\\nTarget Version: ${TARGET_VER}\\nConsole Logs: ${BUILD_URL}console"
}
EOF
                    curl -fsS -X POST "https://api.resend.com/emails" \
                        -H "Authorization: Bearer ${RESEND_API_KEY}" \
                        -H "Content-Type: application/json" \
                        --data-binary @email-payload.json || echo "WARNING: Resend notification failed"
                '''
            }
        }

        always {
            sh '''#!/bin/bash
                rm -f email-payload.json || true
                docker logout "${ECR_REGISTRY}" || true
                docker system prune -af --volumes || true
            '''
            cleanWs()
        }
    }
}
