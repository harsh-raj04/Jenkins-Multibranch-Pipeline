pipeline {
    agent any
    
    // Task 2 (BYOD-2): Pipeline Environment & Credentials (20 Marks)
    environment {
        TF_IN_AUTOMATION = 'true'
        TF_CLI_ARGS = '-no-color'
        // AWS credentials will be injected securely using your existing credential
        AWS_CREDENTIAL = 'Devops-project-id'
        SSH_CRED_ID = 'ssh-private-key'
        PATH = "/usr/local/bin:/usr/bin:/bin:/usr/sbin:/sbin:/opt/homebrew/bin:${env.PATH}"
        // Dynamic variables for BYOD-3
        INSTANCE_IP = ''
        INSTANCE_ID = ''
    }
    
    stages {
        stage('Checkout') {
            steps {
                checkout scm
                echo "Checked out branch: ${env.BRANCH_NAME}"
            }
        }
        
        // Task 3: Initialization & Variable Inspection (20 Marks)
        stage('Terraform Initialization') {
            steps {
                withCredentials([aws(credentialsId: env.AWS_CREDENTIAL)]) {
                    script {
                        echo "Initializing Terraform for branch: ${env.BRANCH_NAME}"
                        sh '/bin/bash -c "terraform init"'
                        
                        // Display the contents of the branch-specific tfvars file
                        echo "Displaying ${env.BRANCH_NAME}.tfvars configuration:"
                        sh """#!/bin/bash
                            if [ -f ${env.BRANCH_NAME}.tfvars ]; then
                                echo "==== Contents of ${env.BRANCH_NAME}.tfvars ===="
                                cat ${env.BRANCH_NAME}.tfvars
                                echo "=============================================="
                            else
                                echo "Warning: ${env.BRANCH_NAME}.tfvars not found!"
                                exit 1
                            fi
                        """
                    }
                }
            }
        }
        
        // Task 4: Branch-Specific Terraform Planning (20 Marks)
        stage('Terraform Plan') {
            steps {
                withCredentials([aws(credentialsId: env.AWS_CREDENTIAL)]) {
                    script {
                        echo "Generating Terraform plan for ${env.BRANCH_NAME} environment"
                        sh """#!/bin/bash
                            terraform plan \
                                -var-file=${env.BRANCH_NAME}.tfvars \
                                -out=${env.BRANCH_NAME}.tfplan
                        """
                        echo "Terraform plan generated successfully!"
                    }
                }
            }
        }
        
        // Task 5: Conditional Manual Approval Gate (20 Marks)
        stage('Validate Apply') {
            when {
                branch 'dev'
            }
            steps {
                script {
                    echo "Requesting approval for dev environment deployment..."
                    def userInput = input(
                        id: 'ApprovalGate',
                        message: 'Do you want to apply this Terraform plan to the dev environment?',
                        parameters: [
                            choice(
                                name: 'APPROVAL',
                                choices: ['Approve', 'Reject'],
                                description: 'Select Approve to proceed with terraform apply'
                            )
                        ]
                    )
                    
                    if (userInput == 'Approve') {
                        echo "Deployment approved! Proceeding to apply..."
                    } else {
                        error "Deployment rejected by user. Aborting pipeline."
                    }
                }
            }
        }
        
        // BYOD-3 Task 1: Provisioning & Output Capture (20 Marks)
        stage('Terraform Apply & Capture Outputs') {
            when {
                branch 'dev'
            }
            steps {
                withCredentials([aws(credentialsId: env.AWS_CREDENTIAL)]) {
                    script {
                        echo "Applying Terraform plan for ${env.BRANCH_NAME} environment"
                        sh """#!/bin/bash
                            terraform apply \
                                -auto-approve \
                                -var-file=${env.BRANCH_NAME}.tfvars
                        """
                        echo "Infrastructure deployed successfully!"
                        
                        // Capture Terraform outputs into environment variables
                        echo "Capturing Terraform outputs..."
                        
                        def instanceIp = sh(
                            script: '/bin/bash -c "terraform output -raw ec2_public_ip 2>/dev/null || echo \'ERROR\'"',
                            returnStdout: true
                        ).trim()
                        
                        def instanceId = sh(
                            script: '/bin/bash -c "terraform output -raw ec2_instance_id 2>/dev/null || echo \'ERROR\'"',
                            returnStdout: true
                        ).trim()
                        
                        if (instanceIp == 'ERROR' || instanceId == 'ERROR') {
                            error "Failed to capture Terraform outputs"
                        }
                        
                        env.INSTANCE_IP = instanceIp
                        env.INSTANCE_ID = instanceId
                        
                        echo "Instance Public IP: ${env.INSTANCE_IP}"
                        echo "Instance ID: ${env.INSTANCE_ID}"
                    }
                }
            }
        }
        
        // BYOD-3 Task 2: Dynamic Inventory Management (20 Marks)
        stage('Create Dynamic Inventory') {
            when {
                branch 'dev'
            }
            steps {
                script {
                    echo "Creating dynamic inventory file for Ansible..."
                    sh """#!/bin/bash
                        cat > dynamic_inventory.ini << EOF
[splunk_servers]
${env.INSTANCE_IP} ansible_user=ubuntu ansible_ssh_private_key_file=~/.ssh/devops.pem ansible_ssh_common_args='-o StrictHostKeyChecking=no'
EOF
                    """
                    
                    echo "Dynamic inventory created:"
                    sh 'cat dynamic_inventory.ini'
                }
            }
        }
        
        // BYOD-3 Task 3: AWS Health Status Verification (20 Marks)
        stage('Wait for Instance Ready') {
            when {
                branch 'dev'
            }
            steps {
                withCredentials([aws(credentialsId: env.AWS_CREDENTIAL)]) {
                    script {
                        echo "Waiting for EC2 instance ${env.INSTANCE_ID} to pass health checks..."
                        sh """#!/bin/bash
                            aws ec2 wait instance-status-ok \
                                --instance-ids ${env.INSTANCE_ID} \
                                --region us-east-1
                        """
                        echo "Instance is healthy and ready!"
                    }
                }
            }
        }
        
        // BYOD-3 Task 4: Splunk Installation & Testing (20 Marks)
        stage('Install Splunk') {
            when {
                branch 'dev'
            }
            steps {
                script {
                    echo "Installing Splunk using Ansible..."
                    ansiblePlaybook(
                        playbook: 'playbooks/splunk.yml',
                        inventory: 'dynamic_inventory.ini',
                        credentialsId: env.SSH_CRED_ID,
                        colorized: true
                    )
                    echo "Splunk installation completed!"
                }
            }
        }
        
        stage('Test Splunk') {
            when {
                branch 'dev'
            }
            steps {
                script {
                    echo "Testing Splunk service..."
                    ansiblePlaybook(
                        playbook: 'playbooks/test-splunk.yml',
                        inventory: 'dynamic_inventory.ini',
                        credentialsId: env.SSH_CRED_ID,
                        colorized: true
                    )
                    echo "Splunk service is active and reachable!"
                }
            }
        }
        
        // Display Outputs
        stage('Show Outputs') {
            when {
                branch 'dev'
            }
            steps {
                withCredentials([aws(credentialsId: env.AWS_CREDENTIAL)]) {
                    script {
                        echo "Displaying Terraform outputs:"
                        sh '/bin/bash -c "terraform output"'
                        echo "\n=== Access Splunk ==="
                        echo "Splunk URL: http://${env.INSTANCE_IP}:8000"
                        echo "Username: admin"
                        echo "Password: Check playbook output or instance"
                    }
                }
            }
        }
        
        // BYOD-3 Task 5: Infrastructure Destruction (20 Marks)
        stage('Validate Destroy') {
            when {
                branch 'dev'
            }
            steps {
                script {
                    echo "Requesting approval for infrastructure destruction..."
                    def destroyInput = input(
                        id: 'DestroyGate',
                        message: 'Do you want to destroy the infrastructure?',
                        parameters: [
                            choice(
                                name: 'DESTROY',
                                choices: ['Skip', 'Destroy'],
                                description: 'Select Destroy to tear down all infrastructure'
                            )
                        ]
                    )
                    
                    if (destroyInput == 'Destroy') {
                        echo "Destruction approved! Proceeding to destroy..."
                        env.DESTROY_APPROVED = 'true'
                    } else {
                        echo "Destruction skipped. Infrastructure will remain active."
                        env.DESTROY_APPROVED = 'false'
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
                    script {
                        echo "Destroying infrastructure for ${env.BRANCH_NAME} environment"
                        sh """#!/bin/bash
                            terraform destroy \
                                -auto-approve \
                                -var-file=${env.BRANCH_NAME}.tfvars
                        """
                        echo "Infrastructure destroyed successfully!"
                    }
                }
            }
        }
    }
    
    post {
        success {
            echo "Pipeline completed successfully for branch: ${env.BRANCH_NAME}"
        }
        failure {
            echo "Pipeline failed for branch: ${env.BRANCH_NAME}"
            script {
                // Auto-destroy on failure for dev branch
                if (env.BRANCH_NAME == 'dev') {
                    echo "Attempting to destroy infrastructure due to pipeline failure..."
                    try {
                        withCredentials([aws(credentialsId: env.AWS_CREDENTIAL)]) {
                            sh """#!/bin/bash
                                cd ${WORKSPACE}
                                terraform destroy \
                                    -auto-approve \
                                    -var-file=${env.BRANCH_NAME}.tfvars || true
                            """
                            echo "Auto-destroy completed"
                        }
                    } catch (Exception e) {
                        echo "Auto-destroy failed: ${e.message}"
                    }
                }
            }
        }
        aborted {
            echo "Pipeline was aborted for branch: ${env.BRANCH_NAME}"
            script {
                // Auto-destroy on abort for dev branch
                if (env.BRANCH_NAME == 'dev') {
                    echo "Attempting to destroy infrastructure due to pipeline abort..."
                    try {
                        withCredentials([aws(credentialsId: env.AWS_CREDENTIAL)]) {
                            sh """#!/bin/bash
                                cd ${WORKSPACE}
                                terraform destroy \
                                    -auto-approve \
                                    -var-file=${env.BRANCH_NAME}.tfvars || true
                            """
                            echo "Auto-destroy completed"
                        }
                    } catch (Exception e) {
                        echo "Auto-destroy failed: ${e.message}"
                    }
                }
            }
        }
        always {
            script {
                // Delete dynamic inventory file
                try {
                    sh 'rm -f dynamic_inventory.ini'
                    echo "Cleaned up dynamic_inventory.ini"
                } catch (Exception e) {
                    echo "Inventory cleanup skipped: ${e.message}"
                }
                
                // Clean workspace
                try {
                    cleanWs(
                        deleteDirs: true,
                        patterns: [
                            [pattern: '*.tfplan', type: 'INCLUDE'],
                            [pattern: '.terraform/', type: 'INCLUDE'],
                            [pattern: 'dynamic_inventory.ini', type: 'INCLUDE']
                        ]
                    )
                } catch (Exception e) {
                    echo "Workspace cleanup skipped: ${e.message}"
                }
            }
        }
    }
}
