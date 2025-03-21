pipeline {
    agent any

    environment {
        DOCKER_USER = 'rekvv'
        IMAGE_NAME = 'backendtest'
        IMAGE_TAG = "${BUILD_NUMBER}"
    }

    stages {

        stage('Git clone') {
            steps {
                echo "Cloning Repository"
                git branch: 'main', url: 'https://github.com/bdt6246/jenkins_backend.git'
            }
        }
//         stage('Gradle Build') {
//             steps {
//                 echo "Add Permission"
//                 sh 'chmod +x gradlew'
//
//                 echo "Build"
//                 sh './gradlew bootJar'
//             }
//         }
//
//         stage('Docker Build') {
//             steps {
//                 script {
//                     def fullImageName = "${DOCKER_USER}/${IMAGE_NAME}:${IMAGE_TAG}"
//                     echo "Building Docker image: ${fullImageName}"
//
//                     docker.build(fullImageName)
//                 }
//             }
//         }
//
//         stage('Docker Push') {
//             steps {
//                 script {
//                     def fullImageName = "${DOCKER_USER}/${IMAGE_NAME}:${IMAGE_TAG}"
//                     echo "Pushing Docker image: ${fullImageName}"
//
//                     docker.withRegistry('https://index.docker.io/v1/', 'DOCKER_HUB') {
//                         docker.image(fullImageName).push()
//                     }
//                 }
//             }
//         }

        stage('Circle CI API') {
            steps {
                script {
                    def TAG_VERSION = "0.1.${BUILD_NUMBER}"

                    withCredentials([
                        string(credentialsId: 'Circle-Token', variable: 'CIRCLECI_TOKEN')
                    ]) {

                        echo "Triggering CircleCI pipeline with TAG_VERSION=${TAG_VERSION}"

                        sh '''
                        curl --location --request POST 'https://circleci.com/api/v2/project/circleci/4JLeNKhG2TrWajvci56PeB/UfMTWLgDF6Q6L58e2omnSn/pipeline/run' \
                        --header "Circle-Token: ${CIRCLECI_TOKEN}" \
                        --header 'Content-Type: application/json' \
                        --data '{
                            "branch": "main",
                            "parameters": {
                                "tag": "'"${TAG_VERSION}"'"
                            }
                        }'
                        '''
                    }
                }
            }
        }


        stage('Select Deployment Color') {
            steps {
                script {
                    def buildNumberInt = BUILD_NUMBER.toInteger()
                    env.COLOR = (buildNumberInt % 2 == 0) ? "green" : "blue"
                    echo "Selected deployment color: ${env.COLOR}"
                }
            }
        }

        stage('K8s Deploy via SSH') {
            steps {
                script {
                    sshPublisher(
                        publishers: [
                            sshPublisherDesc(
                                configName: 'k8s',
                                verbose: true,
                                transfers: [
                                    sshTransfer(
                                        sourceFiles: 'k8s/backend-deployment.yml',
                                        remoteDirectory: '/home/test/k8s',
                                        execCommand: """
                                            echo "=== Replace placeholders in YAML ==="
                                            sed -i 's/__COLOR__/${COLOR}/g' /home/test/k8s/backend-deployment.yml
                                            sed -i 's/latest/${IMAGE_TAG}/g' /home/test/k8s/backend-deployment.yml

                                            echo "=== Deploy to Kubernetes ==="
                                            kubectl apply -f /home/test/k8s/backend-deployment.yml -n khj
                                            kubectl rollout status deployment/backend-deployment-${COLOR} -n khj

                                            echo "=== Switch service to ${COLOR} deployment ==="
                                            kubectl patch service backend-svc -n khj -p '{"spec":{"selector":{"type":"backend","deployment":"${COLOR}"}}}'

                                            echo "=== Scale down previous deployment ==="
                                            if [ "${COLOR}" == "green" ]; then
                                                kubectl scale deployment backend-deployment-blue --replicas=0 -n khj
                                            else
                                                kubectl scale deployment backend-deployment-green --replicas=0 -n khj
                                            fi
                                        """
                                    )
                                ]
                            )
                        ]
                    )
                }
            }
        }
    }
}
