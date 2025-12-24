pipeline {
    agent any

    environment {
        TF_IN_AUTOMATION = 'true'
        TF_CLI_ARGS      = '-no-color'

        AWS_CREDENTIAL   = 'Devops-project-id'
        SSH_CRED_ID      = 'ssh-private-key'

        INSTANCE_IP = ''
        INSTANCE_ID = ''
        DESTROY_APPROVED = 'false'
    }

    stages {

        stage('Checkout') {
            steps {
                checkout scm
                echo "Checked out branch: ${env.BRANCH_NAME}"
            }
        }

        stage('Terraform Initialization') {
            steps {
                withCredentials([aws(credentialsId: env.AWS_CREDENTIAL)]) {
                    withEnv(["PATH=/opt/homebrew/bin:/opt/homebrew/sbin:${env.PATH}"]) {
                        sh 'terraform init'
                        sh 'cat ${BRANCH_NAME}.tfvars'
                    }
                }
            }
        }

        stage('Terraform Plan') {
            steps {
                withCredentials([aws(credentialsId: env.AWS_CREDENTIAL)]) {
                    withEnv(["PATH=/opt/homebrew/bin:/opt/homebrew/sbin:${env.PATH}"]) {
                        sh """
                            terraform plan \
                            -var-file=${BRANCH_NAME}.tfvars \
                            -out=${BRANCH_NAME}.tfplan
                        """
                    }
                }
            }
        }

        stage('Validate Apply') {
            when { branch 'dev' }
            steps {
                input message: 'Apply Terraform plan to DEV?'
            }
        }

        stage('Terraform Apply') {
            when { branch 'dev' }
            steps {
                withCredentials([aws(credentialsId: env.AWS_CREDENTIAL)]) {
                    withEnv(["PATH=/opt/homebrew/bin:/opt/homebrew/sbin:${env.PATH}"]) {
                        sh "terraform apply -auto-approve ${BRANCH_NAME}.tfplan"
                    }
                }
            }
        }

        stage('Capture Terraform Outputs') {
            when { branch 'dev' }
            steps {
                script {
                    withEnv(["PATH=/opt/homebrew/bin:/opt/homebrew/sbin:${env.PATH}"]) {
                        env.INSTANCE_IP = sh(
                            script: 'terraform output -raw ec2_public_ip',
                            returnStdout: true
                        ).trim()

                        env.INSTANCE_ID = sh(
                            script: 'terraform output -raw ec2_instance_id',
                            returnStdout: true
                        ).trim()
                    }

                    echo "Captured IP : ${env.INSTANCE_IP}"
                    echo "Captured ID : ${env.INSTANCE_ID}"

                    if (!env.INSTANCE_IP || !env.INSTANCE_ID) {
                        error "Terraform outputs not captured"
                    }
                }
            }
        }

        stage('Create Dynamic Inventory') {
            when { branch 'dev' }
            steps {
                sh """
                    echo "[splunk_servers]" > dynamic_inventory.ini
                    echo "${INSTANCE_IP} ansible_user=ubuntu ansible_ssh_common_args='-o StrictHostKeyChecking=no'" >> dynamic_inventory.ini
                """
                sh 'cat dynamic_inventory.ini'
            }
        }

        stage('Wait for Instance Ready') {
            when { branch 'dev' }
            steps {
                withCredentials([aws(credentialsId: env.AWS_CREDENTIAL)]) {
                    withEnv(["PATH=/opt/homebrew/bin:/opt/homebrew/sbin:${env.PATH}"]) {
                        sh """
                            aws ec2 wait instance-status-ok \
                            --instance-ids ${INSTANCE_ID} \
                            --region us-east-1
                        """
                    }
                }
            }
        }

        stage('Install Splunk') {
            when { branch 'dev' }
            steps {
                ansiblePlaybook(
                    playbook: 'playbooks/splunk.yml',
                    inventory: 'dynamic_inventory.ini',
                    credentialsId: env.SSH_CRED_ID
                )
            }
        }

        stage('Test Splunk') {
            when { branch 'dev' }
            steps {
                ansiblePlaybook(
                    playbook: 'playbooks/test-splunk.yml',
                    inventory: 'dynamic_inventory.ini',
                    credentialsId: env.SSH_CRED_ID
                )
            }
        }

        stage('Validate Destroy') {
            when { branch 'dev' }
            steps {
                script {
                    def choice = input(
                        message: 'Destroy infrastructure?',
                        parameters: [
                            choice(
                                name: 'ACTION',
                                choices: ['Skip', 'Destroy']
                            )
                        ]
                    )
                    if (choice == 'Destroy') {
                        env.DESTROY_APPROVED = 'true'
                    }
                }
            }
        }

        stage('Terraform Destroy') {
            when {
                allOf {
                    branch 'dev'
                    expression { env.DESTROY_APPROVED == 'true' }
                }
            }
            steps {
                withCredentials([aws(credentialsId: env.AWS_CREDENTIAL)]) {
                    withEnv(["PATH=/opt/homebrew/bin:/opt/homebrew/sbin:${env.PATH}"]) {
                        sh """
                            terraform destroy \
                            -auto-approve \
                            -var-file=${BRANCH_NAME}.tfvars
                        """
                    }
                }
            }
        }
    }

    post {
        always {
            sh 'rm -f dynamic_inventory.ini || true'
        }

        failure {
            script {
                if (env.BRANCH_NAME == 'dev') {
                    withCredentials([aws(credentialsId: env.AWS_CREDENTIAL)]) {
                        withEnv(["PATH=/opt/homebrew/bin:/opt/homebrew/sbin:${env.PATH}"]) {
                            sh """
                                terraform init -reconfigure || true
                                terraform destroy \
                                -auto-approve \
                                -var-file=${BRANCH_NAME}.tfvars || true
                            """
                        }
                    }
                }
            }
        }
    }
}