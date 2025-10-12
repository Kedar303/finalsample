pipeline {
    agent any

    environment {
        DOTNET_PROJECT = "MyApp.csproj"
        BUILD_DIR = "/build_output"
        ANSIBLE_USER = "ansible"
        ANSIBLE_SERVER = "172.31.34.192"
        PLAYBOOK_PATH = "/deploy_iis_app.yml"

        DEV_IIS = "172.31.36.48"
    }

 
    stages {
        stage('Checkout') {
            steps {
                echo "Checking out code..."
                checkout scm
            }
        }

        stage('Build & Publish') {
            steps {
                echo "Building and publishing project..."
                sh "dotnet restore"
                sh "dotnet publish ${DOTNET_PROJECT} -c Release -o ${BUILD_DIR}"
            }
        }

        stage('Versioning') {
            steps {
                script {
                    def version = "v${new Date().format('yyyyMMdd_HHmmss')}_build${env.BUILD_NUMBER}"
                    def archivePath = "/opt/deployments/${version}"   // <— local Groovy variable
                    echo "DEBUG: ARCHIVE_PATH=${archivePath}"

            // Create directory on remote server
                    sh "ssh ${ANSIBLE_USER}@${ANSIBLE_SERVER} 'mkdir -p ${archivePath}'"

            // Copy files to the correct remote path
                    sh "scp -r ${BUILD_DIR}/* ${ANSIBLE_USER}@${ANSIBLE_SERVER}:${archivePath}/"

                    echo "✅ Version archived at: ${archivePath}"
        }
    }
}

 
        stage('Deploy') {
            steps {
                script {
                    // Detect which branch triggered the build
                    if (env.BRANCH_NAME == 'dev') {
                        echo "Deploying to Development IIS (${DEV_IIS})"
                        sh """
                            ssh ${ANSIBLE_USER}@${ANSIBLE_SERVER} \
                            'ansible-playbook ${PLAYBOOK_PATH} \
                            -e target_env=dev -e iis_server=${DEV_IIS} -e version_path=${ARCHIVE_PATH}'
                        """
                    } else if (env.BRANCH_NAME == 'main') {
                        echo "Deploying to Production IIS (${DEV_IIS})"
                        sh """
                            ssh ${ANSIBLE_USER}@${ANSIBLE_SERVER} \
                            'ansible-playbook ${PLAYBOOK_PATH} \
                            -e target_env=prod -e iis_server=${DEV_IIS} -e version_path=${ARCHIVE_PATH}'
                        """
                    } else {
                        echo "Branch ${env.BRANCH_NAME} not configured for deployment."
                    }
                }
            }
        }
    }

 

    post {
        success {
            echo "${env.BRANCH_NAME} deployment completed successfully!"
        }
        
    }
}
