pipeline {
    agent {
        kubernetes {
            label 'wsc-jenkins-agent'
            yamlFile 'jenkins/jenkins-agent.yaml'
        }
    }

    options {
        skipDefaultCheckout(true)
    }

    triggers {
        githubPush()
    }

    environment {
        AWS_REGION = 'ap-northeast-2'
        ECR_REPO = 'wsc-cicd-repo'

        GITHUB_CRED_ID = 'wsc-github-credentials'
        MANIFEST_REPO = 'wsc-app-gitops'
        MAIN_BRANCH = 'main'

        MANIFEST_FILE = 'manifests/rollout.yaml'

        IMAGE_PUSHED = 'false'
        GITOPS_UPDATED = 'false'
    }

    stages {
        stage('Checkout') {
            steps {
                container('jnlp') {
                    checkout scm
                }
            }
        }

        stage('Get AWS Account ID') {
            steps {
                container('aws') {
                    script {
                        def accountId = sh(script: 'aws sts get-caller-identity --query Account --output text', returnStdout: true).trim()

                        env.AWS_ACCOUNT_ID = accountId
                        env.ECR_REGISTRY = "${accountId}.dkr.ecr.${env.AWS_REGION}.amazonaws.com"

                        echo "AWS Account ID: ${env.AWS_ACCOUNT_ID}"
                    }
                }
            }
        }

        stage('Determine Next Image Tag') {
            steps {
                container('aws') {
                    script {
                        def stdout = sh(script: "aws ecr list-images --region ${env.AWS_REGION} --repository-name ${env.ECR_REPO} --query 'imageIds[*].imageTag' --output text", returnStdout: true).trim()

                        if (!stdout || stdout == 'None' || stdout == 'null' || stdout.isEmpty()) {
                            env.IMAGE_TAG = 'v1.0.0'
                        } else {
                            def existingTags = stdout.split(/\s+/)
                            def semverPattern = /^v\d+\.\d+\.\d+$/
                            def versions = existingTags.findAll { it ==~ semverPattern }

                            if (versions) {
                                def maxVersion = [0, 0, 0]

                                versions.each { ver ->
                                    def parts = ver.replaceAll('v', '').split('\\.').collect { it.toInteger() }

                                    for (int i = 0; i < 3; i++) {
                                        if (parts[i] > maxVersion[i]) {
                                            maxVersion = parts
                                            break
                                        } else if (parts[i] < maxVersion[i]) {
                                            break
                                        }
                                    }
                                }

                                maxVersion[2] += 1
                                env.IMAGE_TAG = "v${maxVersion.join('.')}"
                            } else {
                                env.IMAGE_TAG = 'v1.0.0'
                            }
                        }

                        env.DOCKER_IMAGE = "${env.ECR_REGISTRY}/${env.ECR_REPO}:${env.IMAGE_TAG}"

                        echo "Next Docker Image Tag: ${env.IMAGE_TAG}"
                    }
                }
            }
        }

        stage('Wait for Docker') {
            steps {
                container('docker') {
                    sh 'until docker info >/dev/null 2>&1; do echo "Waiting for Docker daemon..."; sleep 2; done'
                }
            }
        }

        stage('ECR Login') {
            steps {
                container('aws') {
                    sh 'aws ecr get-login-password --region ${AWS_REGION} > .ecr-password'
                }

                container('docker') {
                    sh 'cat .ecr-password | docker login --username AWS --password-stdin ${ECR_REGISTRY}'
                }
            }
        }

        stage('Docker Build & Push') {
            steps {
                container('docker') {
                    sh '''
                    docker build -t ${DOCKER_IMAGE} .
                    docker push ${DOCKER_IMAGE}
                    '''
                }

                script {
                    env.IMAGE_PUSHED = 'true'
                }
            }
        }

        stage('Update GitOps Manifest') {
            steps {
                container('jnlp') {
                    withCredentials([usernamePassword(credentialsId: env.GITHUB_CRED_ID, usernameVariable: 'G_USER', passwordVariable: 'G_TOKEN')]) {
                        sh '''
                        rm -rf ${MANIFEST_REPO}
                        git clone --branch ${MAIN_BRANCH} https://${G_USER}:${G_TOKEN}@github.com/${G_USER}/${MANIFEST_REPO}.git

                        cd ${MANIFEST_REPO}

                        sed -i "s|image: .*wsc-cicd-repo:.*|image: ${DOCKER_IMAGE}|g" ${MANIFEST_FILE}

                        git config user.email "jenkins@localhost"
                        git config user.name "jenkins"

                        git add ${MANIFEST_FILE}

                        if ! git diff --cached --quiet; then
                            git commit -m "Deploy application ${IMAGE_TAG}"
                            git push origin ${MAIN_BRANCH}
                        else
                            echo "No changes detected in manifest."
                        fi
                        '''
                    }

                    script {
                        env.GITOPS_UPDATED = 'true'
                    }
                }
            }
        }
    }

    post {
        success {
            echo "Image pushed and GitOps manifest updated: ${env.DOCKER_IMAGE}"
        }

        failure {
            script {
                if (env.IMAGE_PUSHED == 'true' && env.GITOPS_UPDATED != 'true' && env.IMAGE_TAG) {
                    echo "Pipeline failed before GitOps update. Removing image ${env.IMAGE_TAG} from ECR."

                    container('aws') {
                        sh 'aws ecr batch-delete-image --region ${AWS_REGION} --repository-name ${ECR_REPO} --image-ids imageTag=${IMAGE_TAG} || true'
                    }
                }
            }
        }
    }
}