pipeline {
    agent any
    
    environment {
        AWS_ACCESS_KEY_ID     = credentials('aws-access-key-id')
        AWS_SECRET_ACCESS_KEY = credentials('aws-secret-access-key')
        TF_STATE_BUCKET      = 'your-terraform-state-bucket'
        ANSIBLE_INVENTORY    = 'inventory.ini'
        SSH_KEY_FILE         = 'ec2-key.pem'
    }
    
    stages {
        stage('Checkout Code') {
            steps {
                git branch: 'main', url: 'https://github.com/your-username/your-infra-repo.git'
                sh 'mkdir -p terraform ansible'
            }
        }
        
        stage('Terraform Init') {
            steps {
                dir('terraform') {
                    // Create main.tf if not in repo
                    sh '''cat > main.tf <<EOF
provider "aws" {
  region = "us-east-1"
}

resource "aws_instance" "jenkins_target" {
  ami           = "ami-0c55b159cbfafe1f0" # Ubuntu 20.04 LTS
  instance_type = "t2.micro"
  key_name      = "your-key-pair-name"
  tags = {
    Name = "Jenkins-Ansible-Target"
  }
}

output "instance_ip" {
  value = aws_instance.jenkins_target.public_ip
}

output "instance_id" {
  value = aws_instance.jenkins_target.id
}
EOF'''
                    
                    sh 'terraform init -backend-config="bucket=${TF_STATE_BUCKET}"'
                }
            }
        }
        
        stage('Terraform Apply') {
            steps {
                dir('terraform') {
                    sh 'terraform plan -out=tfplan'
                    sh 'terraform apply -auto-approve tfplan'
                    
                    script {
                        // Capture Terraform outputs
                        def tfOutput = sh script: 'terraform output -json', returnStdout: true
                        def outputs = readJSON text: tfOutput
                        env.EC2_IP = outputs.instance_ip.value
                        env.EC2_ID = outputs.instance_id.value
                        
                        // Store IP for Ansible
                        writeFile file: '../ansible/inventory.ini', text: """
                        [target]
                        ${EC2_IP} ansible_user=ubuntu ansible_ssh_private_key_file=../${SSH_KEY_FILE}
                        """
                    }
                }
            }
        }
        
        stage('Wait for SSH') {
            steps {
                script {
                    // Wait until SSH is available
                    def sshReady = false
                    def attempts = 0
                    while (!sshReady && attempts < 30) {
                        try {
                            sh "ssh -o StrictHostKeyChecking=no -i ${SSH_KEY_FILE} ubuntu@${EC2_IP} 'echo SSH ready'"
                            sshReady = true
                        } catch (e) {
                            sleep(10)
                            attempts++
                            echo "Waiting for SSH (attempt ${attempts})..."
                        }
                    }
                    if (!sshReady) {
                        error("SSH never became available")
                    }
                }
            }
        }
        
        stage('Ansible Configuration') {
            steps {
                dir('ansible') {
                    // Create playbook if not in repo
                    sh '''cat > setup.yml <<EOF
---
- hosts: target
  become: yes
  tasks:
    - name: Update apt cache
      apt:
        update_cache: yes

    - name: Install required packages
      apt:
        name: ['python3', 'git', 'curl']
        state: present

    - name: Create demo user
      user:
        name: demo
        groups: sudo
        append: yes
        shell: /bin/bash

    - name: Copy test script
      copy:
        content: |
          #!/bin/bash
          echo "Hello from Jenkins pipeline!"
          echo "Running on $(hostname) at $(date)"
          echo "Instance ID: ${EC2_ID}"
        dest: /home/ubuntu/hello.sh
        mode: '0755'
EOF'''
                    
                    // Run Ansible playbook
                    ansiblePlaybook(
                        playbook: 'setup.yml',
                        inventory: "${ANSIBLE_INVENTORY}",
                        credentialsId: 'ssh-private-key',
                        extras: '--extra-vars "host=target"'
                    )
                }
            }
        }
        
        stage('Run Remote Script') {
            steps {
                sshagent(['ssh-private-key']) {
                    sh """
                    ssh -o StrictHostKeyChecking=no ubuntu@${EC2_IP} '/home/ubuntu/hello.sh'
                    """
                }
            }
        }
    }
    
    post {
        always {
            echo "Pipeline completed with status: ${currentBuild.result}"
            
            // Optional: Terraform destroy
            script {
                if (env.EC2_ID) {
                    echo "EC2 Instance ID: ${EC2_ID}"
                }
                // Uncomment to auto-destroy (not recommended for production)
                // dir('terraform') {
                //     sh 'terraform destroy -auto-approve'
                // }
            }
            
            // Cleanup
            sh 'rm -f ansible/inventory.ini || true'
            cleanWs()
        }
    }
}
