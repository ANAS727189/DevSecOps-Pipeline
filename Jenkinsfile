pipeline {
    agent any

    environment {
        AWS_ACCESS_KEY_ID     = credentials('aws-access-key')
        AWS_SECRET_ACCESS_KEY = credentials('aws-secret-key')
        IMAGE_NAME = "byterunner83/devops-app:${env.BUILD_NUMBER}"
    }

    stages {

        stage('Security Scan (Code)') {
            steps {
                sh "trivy fs . --severity HIGH,CRITICAL --exit-code 1 || true"
            }
        }

        stage('Build & Push Docker Image') {
            steps {
                withCredentials([usernamePassword(
                    credentialsId: 'dockerhub-id',
                    usernameVariable: 'DOCKER_USER',
                    passwordVariable: 'DOCKER_PASS'
                )]) {
                    sh "docker build -t ${IMAGE_NAME} ."
                    sh 'echo $DOCKER_PASS | docker login -u $DOCKER_USER --password-stdin'
                    sh "docker push ${IMAGE_NAME}"
                }
            }
        }

        stage('Security Scan (Image)') {
            steps {
                sh "trivy image ${IMAGE_NAME} --severity HIGH,CRITICAL --exit-code 1 || true"
            }
        }

        stage('Terraform Apply') {
            steps {
                dir('terraform') {
                    sh 'terraform init'
                    sh 'terraform apply -auto-approve'
                    sh 'terraform output -raw public_ip > ../instance_ip.txt'
                }
            }
        }

        stage('Deploy with Ansible') {
            steps {
                script {
                    def IP = readFile('instance_ip.txt').trim()

                    withCredentials([sshUserPrivateKey(
                        credentialsId: 'ec2-ssh-key',
                        keyFileVariable: 'KEY'
                    )]) {
                        sh """
                        export ANSIBLE_HOST_KEY_CHECKING=False
                        ansible-playbook -i ${IP}, \
                        -u ec2-user \
                        --private-key $KEY \
                        --extra-vars 'IMAGE_NAME=${IMAGE_NAME}' \
                        ansible/playbook.yml
                        """
                    }
                }
            }
        }
    }
}