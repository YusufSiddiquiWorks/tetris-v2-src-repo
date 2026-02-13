pipeline {
    // Run on the local machine
    agent any

    environment {
        // --- CONFIGURATION ---
        AWS_REGION = 'ap-south-1'
        
        // SSM Parameter Paths (Where your secrets live in AWS)
        SSM_URL_PATH  = '/ml/docker-registry/url'
        SSM_USER_PATH = '/ml/docker-registry/username'
        SSM_PASS_PATH = '/ml/docker-registry/password'
        
        // REPO_NAME: This is your "username/image-name"
        // Example: 'yusufsiddiqui/my-app'
        REPO_NAME = 'yusufsiddiqui/tetris-v2'

        // --- LOCAL FIXES ---
        // Ensure Jenkins can find 'docker' and 'aws' on your local machine
        PATH = "/usr/local/bin:/usr/bin:/bin:${env.PATH}"
    }

    stages {
        stage('Initialize') {
            steps {
                script {
                    // 1. Get the Short Git Commit SHA (e.g., a1b2c3d)
                    env.GIT_SHA = sh(returnStdout: true, script: "git rev-parse --short HEAD").trim()
                    echo "Starting build for Commit SHA: ${env.GIT_SHA}"
                }
            }
        }

        stage('Fetch Secrets & Build') {
            steps {
                // Inject AWS Keys temporarily so we can talk to SSM
                withCredentials([usernamePassword(credentialsId: 'my-aws-keys', passwordVariable: 'AWS_SECRET_ACCESS_KEY', usernameVariable: 'AWS_ACCESS_KEY_ID')]) {
                    script {
                        echo "Fetching registry credentials from AWS SSM..."
                        
                        // 1. Fetch Registry URL & Username (Standard text)
                        env.REGISTRY_URL = sh(script: "aws ssm get-parameter --name ${SSM_URL_PATH} --region ${AWS_REGION} --query Parameter.Value --output text", returnStdout: true).trim()
                        
                        // 2. Fetch Password (Decrypted SecureString)
                        env.REGISTRY_USER = sh(script: "aws ssm get-parameter --name ${SSM_USER_PATH} --with-decryption --region ${AWS_REGION} --query Parameter.Value --output text", returnStdout: true).trim()
                        env.REGISTRY_PASS = sh(script: "aws ssm get-parameter --name ${SSM_PASS_PATH} --with-decryption --region ${AWS_REGION} --query Parameter.Value --output text", returnStdout: true).trim()

                        // 3. Construct Image Tags
                        // Full Tag: registry/repo-name:git-sha
                        env.IMAGE_SHA    = "${env.REGISTRY_URL}/${env.REPO_NAME}:${env.GIT_SHA}"
                        env.IMAGE_LATEST = "${env.REGISTRY_URL}/${env.REPO_NAME}:latest"

                        echo "Building Image: ${env.IMAGE_SHA}"
                        
                        // 4. Build the Docker Image
                        sh "docker build -t ${env.IMAGE_SHA} -t ${env.IMAGE_LATEST} ."
                    }
                }
            }
        }

        stage('Push to Registry') {
            steps {
                script {
                    // We use 'set +x' to stop Jenkins from printing the password in the logs
                    sh """
                        set +x
                        echo ${env.REGISTRY_PASS} | docker login -u ${env.REGISTRY_USER} --password-stdin ${env.REGISTRY_URL}
                        set -x
                    """
                    
                    echo "Pushing images..."
                    sh "docker push ${env.IMAGE_SHA}"
                    sh "docker push ${env.IMAGE_LATEST}"
                }
            }
        }
    }

    post {
        always {
            script {
                echo 'Cleaning up local images...'
                // Logout and remove images to prevent disk issues
                sh "docker logout ${env.REGISTRY_URL} || true"
                sh "docker rmi ${env.IMAGE_SHA} || true"
                sh "docker rmi ${env.IMAGE_LATEST} || true"
            }
        }
    }
}