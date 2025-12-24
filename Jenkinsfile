pipeline {
    agent any

    environment {
        TF_IN_AUTOMATION = 'true'
        TF_INPUT = 'false'
        TF_CLI_ARGS = '-no-color'
        ANSIBLE_HOST_KEY_CHECKING = 'False'
        AWS_REGION = 'us-east-1'
        
        AWS_CREDENTIAL = 'Devops-project-id'
        SSH_CRED_ID = 'ssh-private-key'
    }

    stages {

        stage('Checkout') {
            steps {
                checkout scm
                echo "Checked out branch: ${env.BRANCH_NAME}"
            }
        }

        stage('Terraform Init') {
            when {
                branch 'dev'
            }
            steps {
                script {
                    withCredentials([aws(credentialsId: env.AWS_CREDENTIAL)]) {
                        sh '''#!/bin/bash
                            export PATH="/opt/homebrew/bin:/usr/local/bin:/usr/bin:/bin:/usr/sbin:/sbin"
                            
                            echo "Initializing Terraform..."
                            terraform init
                            
                            echo "==== Dev environment configuration ===="
                            cat dev.tfvars
                            echo "======================================="
                        '''
                    }
                }
            }
        }

        stage('Terraform Plan') {
            when {
                branch 'dev'
            }
            steps {
                script {
                    withCredentials([aws(credentialsId: env.AWS_CREDENTIAL)]) {
                        sh '''#!/bin/bash
                            export PATH="/opt/homebrew/bin:/usr/local/bin:/usr/bin:/bin:/usr/sbin:/sbin"
                            
                            echo "Generating Terraform plan..."
                            terraform plan -var-file=dev.tfvars -out=dev.tfplan
                        '''
                    }
                }
            }
        }

        stage('Approve Deploy') {
            when {
                branch 'dev'
            }
            steps {
                input message: 'Apply Terraform plan to DEV environment?', ok: 'Deploy'
            }
        }

        stage('Terraform Apply') {
            when {
                branch 'dev'
            }
            steps {
                script {
                    withCredentials([aws(credentialsId: env.AWS_CREDENTIAL)]) {
                        sh '''#!/bin/bash
                            export PATH="/opt/homebrew/bin:/usr/local/bin:/usr/bin:/bin:/usr/sbin:/sbin"
                            
                            echo "Applying Terraform plan..."
                            terraform apply -auto-approve dev.tfplan
                            echo "✓ Infrastructure deployed!"
                        '''
                    }
                }
            }
        }

        stage('Capture Outputs') {
            when {
                branch 'dev'
            }
            steps {
                script {
                    env.INSTANCE_ID = sh(
                        script: '''#!/bin/bash
                            export PATH="/opt/homebrew/bin:/usr/local/bin:/usr/bin:/bin:/usr/sbin:/sbin"
                            terraform output -raw ec2_instance_id
                        ''',
                        returnStdout: true
                    ).trim()

                    env.INSTANCE_IP = sh(
                        script: '''#!/bin/bash
                            export PATH="/opt/homebrew/bin:/usr/local/bin:/usr/bin:/bin:/usr/sbin:/sbin"
                            terraform output -raw ec2_public_ip
                        ''',
                        returnStdout: true
                    ).trim()

                    echo "Instance ID: ${env.INSTANCE_ID}"
                    echo "Instance IP: ${env.INSTANCE_IP}"

                    if (!env.INSTANCE_IP || !env.INSTANCE_ID) {
                        error "Failed to capture outputs"
                    }
                }
            }
        }

        stage('Create Inventory') {
            when {
                branch 'dev'
            }
            steps {
                sh '''#!/bin/bash
                    export PATH="/opt/homebrew/bin:/usr/local/bin:/usr/bin:/bin:/usr/sbin:/sbin"
                    
                    cat > dynamic_inventory.ini <<EOF
[splunk_servers]
${INSTANCE_IP} ansible_user=ubuntu ansible_ssh_private_key_file=${HOME}/.ssh/devops.pem ansible_ssh_common_args='-o StrictHostKeyChecking=no'
EOF
                    
                    echo "==== Ansible Inventory ===="
                    cat dynamic_inventory.ini
                    echo "==========================="
                '''
            }
        }

        stage('Wait for Instance') {
            when {
                branch 'dev'
            }
            steps {
                script {
                    withCredentials([aws(credentialsId: env.AWS_CREDENTIAL)]) {
                        sh '''#!/bin/bash
                            export PATH="/opt/homebrew/bin:/usr/local/bin:/usr/bin:/bin:/usr/sbin:/sbin"
                            
                            echo "Waiting for instance to be ready..."
                            aws ec2 wait instance-status-ok --instance-ids ${INSTANCE_ID} --region ${AWS_REGION}
                            echo "✓ Instance is ready!"
                        '''
                    }
                }
            }
        }

        stage('Install Splunk') {
            when {
                branch 'dev'
            }
            steps {
                script {
                    withCredentials([
                        sshUserPrivateKey(
                            credentialsId: 'ssh-private-key',
                            keyFileVariable: 'SSH_KEY',
                            usernameVariable: 'SSH_USER'
                        )
                    ]) {
                        sh '''#!/bin/bash
                            export PATH="/opt/homebrew/bin:/usr/local/bin:/usr/bin:/bin:/usr/sbin:/sbin"
                            
                            chmod 600 ${SSH_KEY}
                            ansible-playbook \
                                -i dynamic_inventory.ini \
                                --private-key ${SSH_KEY} \
                                playbooks/splunk.yml
                        '''
                    }
                }
            }
        }

        stage('Test Splunk') {
            when {
                branch 'dev'
            }
            steps {
                script {
                    withCredentials([
                        sshUserPrivateKey(
                            credentialsId: env.SSH_CRED_ID,
                            keyFileVariable: 'SSH_KEY',
                            usernameVariable: 'SSH_USER'
                        )
                    ]) {
                        sh '''#!/bin/bash
                            export PATH="/opt/homebrew/bin:/usr/local/bin:/usr/bin:/bin:/usr/sbin:/sbin"
                            
                            chmod 600 ${SSH_KEY}
                            ansible-playbook \
                                -i dynamic_inventory.ini \
                                --private-key ${SSH_KEY} \
                                playbooks/test-splunk.yml
                        '''
                    }
                }
            }
        }

        stage('Show Results') {
            when {
                branch 'dev'
            }
            steps {
                sh '''#!/bin/bash
                    export PATH="/opt/homebrew/bin:/usr/local/bin:/usr/bin:/bin:/usr/sbin:/sbin"
                    
                    echo "=========================================="
                    echo "       DEPLOYMENT SUCCESSFUL!"
                    echo "=========================================="
                    terraform output
                    echo ""
                    echo "Splunk Web: http://${INSTANCE_IP}:8000"
                    echo "  Username: admin"
                    echo "  Password: SplunkAdmin123!"
                    echo ""
                    echo "Management: ${INSTANCE_IP}:8089"
                    echo "=========================================="
                '''
            }
        }

        stage('Approve Destroy') {
            when {
                branch 'dev'
            }
            steps {
                input message: 'Destroy infrastructure?', ok: 'Yes, Destroy'
            }
        }

        stage('Terraform Destroy') {
            when {
                branch 'dev'
            }
            steps {
                script {
                    withCredentials([aws(credentialsId: env.AWS_CREDENTIAL)]) {
                        sh '''#!/bin/bash
                            export PATH="/opt/homebrew/bin:/usr/local/bin:/usr/bin:/bin:/usr/sbin:/sbin"
                            
                            terraform destroy -auto-approve -var-file=dev.tfvars
                            echo "✓ Infrastructure destroyed"
                        '''
                    }
                }
            }
        }
    }

    post {
        always {
            sh '''#!/bin/bash
                export PATH="/opt/homebrew/bin:/usr/local/bin:/usr/bin:/bin:/usr/sbin:/sbin"
                rm -f dynamic_inventory.ini dev.tfplan || true
            '''
        }

        failure {
            script {
                echo "Pipeline failed - destroying infrastructure..."
                withCredentials([aws(credentialsId: env.AWS_CREDENTIAL)]) {
                    sh '''#!/bin/bash
                        export PATH="/opt/homebrew/bin:/usr/local/bin:/usr/bin:/bin:/usr/sbin:/sbin"
                        
                        terraform init -reconfigure || true
                        terraform destroy -auto-approve -var-file=dev.tfvars || true
                    '''
                }
            }
        }

        aborted {
            script {
                echo "Pipeline aborted - destroying infrastructure..."
                withCredentials([aws(credentialsId: env.AWS_CREDENTIAL)]) {
                    sh '''#!/bin/bash
                        export PATH="/opt/homebrew/bin:/usr/local/bin:/usr/bin:/bin:/usr/sbin:/sbin"
                        
                        terraform init -reconfigure || true
                        terraform destroy -auto-approve -var-file=dev.tfvars || true
                    '''
                }
            }
        }

        success {
            echo "✅ BYOD-3 Pipeline completed successfully!"
        }
    }
}
