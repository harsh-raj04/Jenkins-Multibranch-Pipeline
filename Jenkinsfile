pipeline {
    agent any

    environment {
        TF_IN_AUTOMATION = 'true'
        TF_INPUT = 'false'
        ANSIBLE_HOST_KEY_CHECKING = 'False'
        AWS_REGION = 'us-east-1'
        
        // Fix PATH for macOS Jenkins
        PATH = "/opt/homebrew/bin:/opt/homebrew/sbin:/usr/local/bin:/usr/bin:/bin:/usr/sbin:/sbin"
    }

    stages {

        // =====================================================
        // STAGE 1: CHECKOUT
        // =====================================================
        stage('📥 Checkout') {
            steps {
                checkout scm
                echo "Checked out branch: ${env.BRANCH_NAME}"
            }
        }

        // =====================================================
        // STAGE 2: TERRAFORM INIT
        // =====================================================
        stage('🔧 Terraform Init') {
            when {
                branch 'dev'
            }
            steps {
                withCredentials([
                    string(credentialsId: 'AWS_ACCESS_KEY_ID', variable: 'AWS_ACCESS_KEY_ID'),
                    string(credentialsId: 'AWS_SECRET_ACCESS_KEY', variable: 'AWS_SECRET_ACCESS_KEY')
                ]) {
                    sh '''
                      export AWS_ACCESS_KEY_ID
                      export AWS_SECRET_ACCESS_KEY
                      terraform init
                      echo "==== Contents of ${BRANCH_NAME}.tfvars ===="
                      cat ${BRANCH_NAME}.tfvars
                      echo "=============================================="
                    '''
                }
            }
        }

        // =====================================================
        // STAGE 3: TERRAFORM PLAN
        // =====================================================
        stage('📋 Terraform Plan') {
            when {
                branch 'dev'
            }
            steps {
                withCredentials([
                    string(credentialsId: 'AWS_ACCESS_KEY_ID', variable: 'AWS_ACCESS_KEY_ID'),
                    string(credentialsId: 'AWS_SECRET_ACCESS_KEY', variable: 'AWS_SECRET_ACCESS_KEY')
                ]) {
                    sh '''
                      export AWS_ACCESS_KEY_ID
                      export AWS_SECRET_ACCESS_KEY
                      
                      terraform plan \
                        -var-file=${BRANCH_NAME}.tfvars \
                        -out=${BRANCH_NAME}.tfplan
                    '''
                }
            }
        }

        // =====================================================
        // STAGE 4: VALIDATE APPLY
        // =====================================================
        stage('✅ Validate Apply') {
            when {
                branch 'dev'
            }
            steps {
                input message: 'Apply Terraform plan to DEV environment?',
                      ok: 'Yes, Deploy Infrastructure'
            }
        }

        // =====================================================
        // STAGE 5: TERRAFORM APPLY
        // =====================================================
        stage('🚀 Terraform Apply') {
            when {
                branch 'dev'
            }
            steps {
                withCredentials([
                    string(credentialsId: 'AWS_ACCESS_KEY_ID', variable: 'AWS_ACCESS_KEY_ID'),
                    string(credentialsId: 'AWS_SECRET_ACCESS_KEY', variable: 'AWS_SECRET_ACCESS_KEY')
                ]) {
                    sh '''
                      export AWS_ACCESS_KEY_ID
                      export AWS_SECRET_ACCESS_KEY

                      terraform apply -auto-approve ${BRANCH_NAME}.tfplan
                      echo "Infrastructure deployed successfully!"
                    '''
                }
            }
        }

        // =====================================================
        // STAGE 6: CAPTURE OUTPUTS
        // =====================================================
        stage('📤 Capture Terraform Outputs') {
            when {
                branch 'dev'
            }
            steps {
                script {
                    env.INSTANCE_ID = sh(
                        script: "terraform output -raw ec2_instance_id",
                        returnStdout: true
                    ).trim()

                    env.INSTANCE_IP = sh(
                        script: "terraform output -raw ec2_public_ip",
                        returnStdout: true
                    ).trim()

                    echo "✓ Captured Instance ID: ${env.INSTANCE_ID}"
                    echo "✓ Captured Instance IP: ${env.INSTANCE_IP}"
                    
                    if (!env.INSTANCE_IP || !env.INSTANCE_ID) {
                        error "Failed to capture Terraform outputs"
                    }
                }
            }
        }

        // =====================================================
        // STAGE 7: DYNAMIC INVENTORY
        // =====================================================
        stage('📄 Create Dynamic Inventory') {
            when {
                branch 'dev'
            }
            steps {
                sh '''
                  cat <<EOF > dynamic_inventory.ini
[splunk_servers]
${INSTANCE_IP} ansible_user=ubuntu ansible_ssh_private_key_file=${HOME}/.ssh/devops.pem ansible_ssh_common_args='-o StrictHostKeyChecking=no'
EOF
                  echo "==== Dynamic Inventory ===="
                  cat dynamic_inventory.ini
                  echo "==========================="
                '''
            }
        }

        // =====================================================
        // STAGE 8: AWS HEALTH CHECK
        // =====================================================
        stage('🩺 Wait for Instance Ready') {
            when {
                branch 'dev'
            }
            steps {
                withCredentials([
                    string(credentialsId: 'AWS_ACCESS_KEY_ID', variable: 'AWS_ACCESS_KEY_ID'),
                    string(credentialsId: 'AWS_SECRET_ACCESS_KEY', variable: 'AWS_SECRET_ACCESS_KEY')
                ]) {
                    sh '''
                      export AWS_ACCESS_KEY_ID
                      export AWS_SECRET_ACCESS_KEY

                      echo "Waiting for instance to be ready..."
                      aws ec2 wait instance-status-ok \
                        --instance-ids ${INSTANCE_ID} \
                        --region ${AWS_REGION}
                      echo "✓ Instance is ready!"
                    '''
                }
            }
        }

        // =====================================================
        // STAGE 9: SPLUNK INSTALL
        // =====================================================
        stage('📦 Install Splunk') {
            when {
                branch 'dev'
            }
            steps {
                withCredentials([
                    sshUserPrivateKey(
                        credentialsId: 'ssh-private-key',
                        keyFileVariable: 'SSH_KEY',
                        usernameVariable: 'SSH_USER'
                    )
                ]) {
                    sh '''
                      chmod 600 $SSH_KEY
                      ansible-playbook \
                        -i dynamic_inventory.ini \
                        --private-key $SSH_KEY \
                        playbooks/splunk.yml
                    '''
                }
            }
        }

        // =====================================================
        // STAGE 10: SPLUNK VERIFICATION
        // =====================================================
        stage('✅ Test Splunk') {
            when {
                branch 'dev'
            }
            steps {
                withCredentials([
                    sshUserPrivateKey(
                        credentialsId: 'ssh-private-key',
                        keyFileVariable: 'SSH_KEY',
                        usernameVariable: 'SSH_USER'
                    )
                ]) {
                    sh '''
                      chmod 600 $SSH_KEY
                      ansible-playbook \
                        -i dynamic_inventory.ini \
                        --private-key $SSH_KEY \
                        playbooks/test-splunk.yml
                    '''
                }
            }
        }

        // =====================================================
        // STAGE 11: SHOW OUTPUTS
        // =====================================================
        stage('📊 Show Outputs') {
            when {
                branch 'dev'
            }
            steps {
                sh '''
                  echo "=========================================="
                  echo "      DEPLOYMENT SUCCESSFUL!"
                  echo "=========================================="
                  terraform output
                  echo ""
                  echo "🌐 Splunk Web UI: http://${INSTANCE_IP}:8000"
                  echo "   Username: admin"
                  echo "   Password: SplunkAdmin123!"
                  echo ""
                  echo "🔌 Management Port: ${INSTANCE_IP}:8089"
                  echo "=========================================="
                '''
            }
        }

        // =====================================================
        // STAGE 12: VALIDATE DESTROY
        // =====================================================
        stage('🛑 Validate Destroy') {
            when {
                branch 'dev'
            }
            steps {
                input message: 'Do you want to destroy the infrastructure?',
                      ok: 'Yes, Destroy Everything'
            }
        }

        // =====================================================
        // STAGE 13: TERRAFORM DESTROY
        // =====================================================
        stage('🔥 Terraform Destroy') {
            when {
                branch 'dev'
            }
            steps {
                withCredentials([
                    string(credentialsId: 'AWS_ACCESS_KEY_ID', variable: 'AWS_ACCESS_KEY_ID'),
                    string(credentialsId: 'AWS_SECRET_ACCESS_KEY', variable: 'AWS_SECRET_ACCESS_KEY')
                ]) {
                    sh '''
                      export AWS_ACCESS_KEY_ID
                      export AWS_SECRET_ACCESS_KEY

                      terraform destroy \
                        -auto-approve \
                        -var-file=${BRANCH_NAME}.tfvars
                      
                      echo "✓ Infrastructure destroyed successfully"
                    '''
                }
            }
        }
    }

    // =====================================================
    // POST ACTIONS
    // =====================================================
    post {
        always {
            script {
                echo "🧹 Cleaning up temporary files..."
                sh 'rm -f dynamic_inventory.ini *.tfplan'
            }
        }

        failure {
            script {
                echo "❌ Pipeline failed — attempting to destroy infrastructure..."
                withCredentials([
                    string(credentialsId: 'AWS_ACCESS_KEY_ID', variable: 'AWS_ACCESS_KEY_ID'),
                    string(credentialsId: 'AWS_SECRET_ACCESS_KEY', variable: 'AWS_SECRET_ACCESS_KEY')
                ]) {
                    sh '''
                      export AWS_ACCESS_KEY_ID
                      export AWS_SECRET_ACCESS_KEY
                      
                      terraform init -reconfigure || true
                      terraform destroy -auto-approve -var-file=${BRANCH_NAME}.tfvars || true
                    '''
                }
            }
        }

        aborted {
            script {
                echo "⚠️ Pipeline aborted — attempting to destroy infrastructure..."
                withCredentials([
                    string(credentialsId: 'AWS_ACCESS_KEY_ID', variable: 'AWS_ACCESS_KEY_ID'),
                    string(credentialsId: 'AWS_SECRET_ACCESS_KEY', variable: 'AWS_SECRET_ACCESS_KEY')
                ]) {
                    sh '''
                      export AWS_ACCESS_KEY_ID
                      export AWS_SECRET_ACCESS_KEY
                      
                      terraform init -reconfigure || true
                      terraform destroy -auto-approve -var-file=${BRANCH_NAME}.tfvars || true
                    '''
                }
            }
        }

        success {
            echo "✅ BYOD-3 Pipeline completed successfully!"
        }
    }
}
