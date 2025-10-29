pipeline {
    agent any

    environment {
        DOTNET_PROJECT = "webapp.csproj"  //change name to .cs.proj as per the project
        BUILD_DIR = "/build_output" //here build files will be stored
        ANSIBLE_USER = "ec2-user" //login user in ansible server, password less auth
        ANSIBLE_SERVER = "172.31.34.192"
        PLAYBOOK_PATH = "/deploy_iis_app.yml" //playbook path in ansible server
        PROD_IIS = "172.31.15.224"
        DEV_IIS = "172.31.2.240"
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
                    env.ARCHIVE_PATH = "/opt/deployments/${version}"  
                    echo "DEBUG: ARCHIVE_PATH=${env.ARCHIVE_PATH}"

                    // Create directory on remote server
                    sh "ssh ${ANSIBLE_USER}@${ANSIBLE_SERVER} 'mkdir -p ${env.ARCHIVE_PATH}'" //create directory manually and assign permissions manually to the ansible user to that folder

                    // Copy files to the correct remote path
                    sh "scp -r ${BUILD_DIR}/* ${ANSIBLE_USER}@${ANSIBLE_SERVER}:${env.ARCHIVE_PATH}/"

                    echo "Version archived at: ${env.ARCHIVE_PATH}"
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
                            'ansible-playbook ${PLAYBOOK_PATH} -i /hosts -l windows_dev \
                            -e target_env=dev -e iis_server=${DEV_IIS} -e version_path=${env.ARCHIVE_PATH}'
                        """
                    } else if (env.BRANCH_NAME == 'main') {
                        echo "Deploying to Production IIS (${PROD_IIS})"
                        sh """
                            ssh ${ANSIBLE_USER}@${ANSIBLE_SERVER} \
                            'ansible-playbook ${PLAYBOOK_PATH} -i /hosts -l windows_prod \
                            -e target_env=prod -e iis_server=${PROD_IIS} -e version_path=${env.ARCHIVE_PATH}'
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
        failure {
            echo "Deployment failed for branch ${env.BRANCH_NAME}"
        }
    }
}
