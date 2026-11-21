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
            name: 'RELEASE_VERSION',
            defaultValue: '2.3.0',
            description: 'Semantic version format only (example: 2.3.0)'
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
                    env.APP_VERSION = (params.RELEASE_VERSION ?: '2.3.0').trim()

                    if (!(env.APP_VERSION ==~ /^[0-9]+\.[0-9]+\.[0-9]+$/)) {
                        error("Invalid version '${env.APP_VERSION}'! Use Semantic Versioning like 2.3.0")
                    }

                    env.ARTIFACT_VERSION = "${env.APP_VERSION}-build-${env.BUILD_NUMBER}-${env.GIT_SHORT_SHA}"
                    env.JAR_FILENAME     = "java-microservice-${env.ARTIFACT_VERSION}.jar"
                    env.S3_ARTIFACT_URI  = "s3://${env.S3_BUCKET}/java-microservice/${env.APP_VERSION}/build-${env.BUILD_NUMBER}/${env.JAR_FILENAME}"
                    env.ECR_IMAGE_TAG    = "${env.APP_VERSION}-build-${env.BUILD_NUMBER}"

                    echo "App Version      : ${env.APP_VERSION}"
                    echo "S3 Artifact URI  : ${env.S3_ARTIFACT_URI}"
                    echo "ECR Image Tag    : ${env.ECR_IMAGE_TAG}"
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

                    SOURCE_JAR="target/java-microservice-${APP_VERSION}.jar"
                    test -f "${SOURCE_JAR}"
                    cp "${SOURCE_JAR}" "target/${JAR_FILENAME}"
                    ls -lh "target/${JAR_FILENAME}"
                '''
            }
        }

        stage('Upload Versioned JAR to S3') {
            steps {
                sh '''#!/bin/bash
                    set -euo pipefail

                    cat > build-info.txt <<EOF
App Version: ${APP_VERSION}
Build Number: ${BUILD_NUMBER}
Git Commit: ${GIT_FULL_SHA}
ECR Image: ${ECR_REGISTRY}/${ECR_REPOSITORY}:${ECR_IMAGE_TAG}
S3 Artifact: ${S3_ARTIFACT_URI}
Build Date: $(date -u +"%Y-%m-%dT%H:%M:%SZ")
EOF

                    aws s3 cp "target/${JAR_FILENAME}" "${S3_ARTIFACT_URI}" \
                        --region "${AWS_REGION}" \
                        --expected-bucket-owner "${AWS_ACCOUNT_ID}" \
                        --sse AES256 \
                        --only-show-errors

                    aws s3 cp build-info.txt "s3://${S3_BUCKET}/java-microservice/${APP_VERSION}/build-${BUILD_NUMBER}/build-info.txt" \
                        --region "${AWS_REGION}" \
                        --expected-bucket-owner "${AWS_ACCOUNT_ID}" \
                        --sse AES256 \
                        --only-show-errors

                    aws s3 ls "s3://${S3_BUCKET}/java-microservice/${APP_VERSION}/build-${BUILD_NUMBER}/" --region "${AWS_REGION}"
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
                        -t "${ECR_REGISTRY}/${ECR_REPOSITORY}:${ARTIFACT_VERSION}" \
                        .

                    docker push "${ECR_REGISTRY}/${ECR_REPOSITORY}:${ECR_IMAGE_TAG}"
                    docker push "${ECR_REGISTRY}/${ECR_REPOSITORY}:${ARTIFACT_VERSION}"

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
            echo "CI Pipeline PASSED | Version: ${env.APP_VERSION} | S3: ${env.S3_ARTIFACT_URI} | ECR: ${env.ECR_REGISTRY}/${env.ECR_REPOSITORY}:${env.ECR_IMAGE_TAG}"
        }
        failure {
            echo "CI Pipeline FAILED | Check console logs. Ephemeral agent will auto-terminate."
        }
        always {
            sh '''#!/bin/bash
                docker logout "${ECR_REGISTRY}" || true
                docker system prune -af --volumes || true
            '''
            cleanWs()
        }
    }
}
