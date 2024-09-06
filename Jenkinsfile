pipeline {
    agent any
    environment {
        APP_NAME="phonebook"
        APP_REPO_NAME="clarusway-repo/${APP_NAME}-app"
        AWS_ACCOUNT_ID=sh(script:'aws sts get-caller-identity --query Account --output text', returnStdout:true).trim()
        AWS_REGION="us-east-1"
        ECR_REGISTRY="${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com"
        }

        stages {
            stage('Create ECR Repo') {
                steps {
                    echo "Creating ECR Repo for ${APP_NAME}-app"
                    sh '''
                    aws ecr describe-repositories --region ${AWS_REGION} --repository-name ${APP_REPO_NAME} || \
                            aws ecr create-repository \
                            --repository-name ${APP_REPO_NAME} \
                            --image-scanning-configuration scanOnPush=true \
                            --image-tag-mutability MUTABLE \
                            --region ${AWS_REGION}
                    '''
                        }
                    }

            stage('Build App Docker Images') {
                steps {
                    echo 'Building App Images for app'
                    sh '''
                    aws ecr get-login-password --region ${AWS_REGION} | docker login --username AWS --password-stdin ${ECR_REGISTRY}
                    cd image_for_web_server
                    docker build -t ${ECR_REGISTRY}/${APP_REPO_NAME}:web .
                    docker push ${ECR_REGISTRY}/${APP_REPO_NAME}:web
                    cd ../image_for_result_server
                    docker build -t ${ECR_REGISTRY}/${APP_REPO_NAME}:result .
                    docker push ${ECR_REGISTRY}/${APP_REPO_NAME}:result
                    docker image ls
                    '''
                }
            }

            stage('build k8s cluster') {
                agent {
                    docker { image 'olivercw/tf-agent' }
                }
                steps {
                    echo 'build infra with tf'
                    sh '''
                    cd create-k8s-cluster
                    terraform init
                    terraform apply --auto-approve
                    '''
                }
            }

            stage('wait 200') {
                steps {
                    sh 'sleep 200'
                }
            }

            stage('Deploy the App') {
                environment {
                    MYKEY     = credentials('project-207')
                }
                steps {
                    sh 'ansible-playbook playbook.yml --private-key $MYKEY'
                }
        }

    }

}