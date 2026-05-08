pipeline {
    agent any

    environment {
        DOCKERHUB_USER  = 'andresiglesiasbarbara'
        DOCKERHUB_CREDS = 'dockerhub-credentials'
        GITOPS_REPO     = 'git@github.com:andresiglesiasb/example-voting-app-gitops.git'
        GITOPS_CREDS    = 'github-gitops-ssh'
        VERSION         = "v1.0.${BUILD_NUMBER}"
    }

    stages {

        stage('vote') {
            steps {
                script {
                    docker.withRegistry('', DOCKERHUB_CREDS) {
                        def img = docker.build("${DOCKERHUB_USER}/vote:${VERSION}", "./vote")
                        img.push()
                    }
                    sh "docker rmi ${DOCKERHUB_USER}/vote:${VERSION}"
                }
            }
        }

        stage('result') {
            steps {
                script {
                    docker.withRegistry('', DOCKERHUB_CREDS) {
                        def img = docker.build("${DOCKERHUB_USER}/result:${VERSION}", "./result")
                        img.push()
                    }
                    sh "docker rmi ${DOCKERHUB_USER}/result:${VERSION}"
                }
            }
        }

        stage('worker') {
            steps {
                script {
                    docker.withRegistry('', DOCKERHUB_CREDS) {
                        def img = docker.build("${DOCKERHUB_USER}/worker:${VERSION}", "./worker")
                        img.push()
                    }
                    sh "docker rmi ${DOCKERHUB_USER}/worker:${VERSION}"
                }
            }
        }

        stage('Update GitOps repo') {
            steps {
                sshagent([GITOPS_CREDS]) {
                    sh """
                        rm -rf gitops-tmp
                        GIT_SSH_COMMAND="ssh -o StrictHostKeyChecking=no" git clone ${GITOPS_REPO} gitops-tmp
                        cd gitops-tmp

                        sed -i 's|image: ${DOCKERHUB_USER}/vote:.*|image: ${DOCKERHUB_USER}/vote:${VERSION}|'     vote-deployment.yaml
                        sed -i 's|image: ${DOCKERHUB_USER}/result:.*|image: ${DOCKERHUB_USER}/result:${VERSION}|' result-deployment.yaml
                        sed -i 's|image: ${DOCKERHUB_USER}/worker:.*|image: ${DOCKERHUB_USER}/worker:${VERSION}|' worker-deployment.yaml

                        git config user.email "jenkins@devops-lab"
                        git config user.name "Jenkins"
                        git add vote-deployment.yaml result-deployment.yaml worker-deployment.yaml
                        git commit -m "ci: bump image tags to ${VERSION} [build #${BUILD_NUMBER}]"
                        GIT_SSH_COMMAND="ssh -o StrictHostKeyChecking=no" git push origin main
                    """
                }
            }
        }
    }

    post {
        always {
            sh "rm -rf gitops-tmp"
            sh "docker system prune -f --filter 'until=1h' || true"
            cleanWs()
        }
    }
}
