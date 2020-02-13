pipeline {
   agent any
   stages {
        stage('Select docker registry') {
                when{
                    expression { env.DEPLOY == 'true' || env.PUSH_REGISTRY == 'true' }
                }
                steps {
                         script {
                           def REGISTRY_LIST = []
                           def buildStages = env.getEnvironment()
                           for (builds in buildStages) {
                               if(builds.key.startsWith("REGISTRY_")) {
                                 REGISTRY_LIST.add(builds.value)
                               }
                           }
                           env.select_docker_registry = input  message: 'Select docker registry ',ok : 'Proceed',id :'tag_id',
                           parameters:[choice(choices: REGISTRY_LIST, description: 'Select docker registry', name: 'dockerregistry')]
                           echo "Selected Registry is ${env.select_docker_registry}"
                            env.registryCredId="REGISTRY_"+env.select_docker_registry
                         }
                }
        }
        stage('Select Environment') {
                    when{
                        expression { env.DEPLOY == 'true' }
                    }
                    steps {
                              script {
                                def SERVER_LIST=[]
                                def buildStages = env.getEnvironment()
                                for (builds in buildStages) {
                                    if(builds.key.startsWith("SERVER_")) {
                                      SERVER_LIST.add(builds.value)
                                    }
                                }
                                env.selected_environment = input  message: 'Select environment ',ok : 'Proceed',id :'tag_id',
                                parameters:[choice(choices: SERVER_LIST, description: 'Select environment', name: 'env')]
                                echo "Deploying ${env.selected_environment}."
                                env.credid = "SSH_"+env.selected_environment
                              }
                    }
        }
        stage('Checkout') {
             when {
               expression { env.CHECKOUT == 'true' }
             }
             steps {
                git branch: "${BRANCH}", credentialsId:'service', url:''
             }
        }
        stage('Debug parameters'){
            steps {
                echo "selected ${BRANCH}"
            }
        }
        stage('Build Image') {
             when {
                   expression { env.BUILD_IMAGE == 'true' }
             }
             steps {
                sh """
                    cd mongodb
                    pwd
                    docker build -t mongodb .
                    cd ..
                """
            }
        }
        stage('Push Registry') {
              when {
                 expression { env.PUSH_REGISTRY == 'true' }
              }
              steps {
                  withCredentials([usernamePassword(credentialsId: env.registryCredId, passwordVariable: 'REGISTRY_PASSWORD', usernameVariable: 'REGISTRY_USERNAME')]) {
                                                   sh """
                                                                              docker login -u ${REGISTRY_USERNAME} -p ${REGISTRY_PASSWORD} ${env.select_docker_registry}
                                                                              docker tag mongodb:latest ${env.select_docker_registry}/speedy/mongodb:latest
                                                                              docker push ${env.select_docker_registry}/speedy/mongodb:latest
                                                                              docker logout ${env.select_docker_registry}
                                                   """
                }
              }
        }
        stage('Deploy') {
            when{
                  expression { env.DEPLOY == 'true' }
            }
            steps {
                    withCredentials([usernamePassword(credentialsId: env.credid , passwordVariable: 'SSH_PASSWORD', usernameVariable: 'SSH_USERNAME')]) {
                             sh """
                                 > /tmp/processor_hosts
                                 echo "[server]" > /tmp/mongodb_hosts
                                 echo " ${env.selected_environment}" >> /tmp/mongodb_hosts
                                 cd ansible_deployment/
                                 ansible-playbook -i /tmp/mongodb_hosts playbooks/deploy.yaml -e "selected_server=${env.selected_environment}" -e "selected_registry=${env.select_docker_registry}" --extra-vars 'ansible_ssh_pass=${SSH_PASSWORD}' --extra-vars='ansible_ssh_user=${SSH_USERNAME}'
                             """
                    }
            }
        }
    }
}
