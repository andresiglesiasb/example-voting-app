pipeline {
    agent any

    environment {
        DOCKERHUB_USER  = 'andresiglesiasbarbara'
        DOCKERHUB_CREDS = 'dockerhub-credentials'           // Credential ID from step A1
        GITOPS_REPO     = 'git@github.com:andresiglesiasb/example-voting-app-gitops.git'
        GITOPS_CREDS    = 'github-gitops-ssh'               // Credential ID from step A2
        VERSION         = "v1.0.${BUILD_NUMBER}"            // e.g. v1.0.1, v1.0.2 ...
    }

    stages {

        stage('Build images') {
            steps {
                script {
                    docker.build("${DOCKERHUB_USER}/vote:${VERSION}",   "./vote")
                    docker.build("${DOCKERHUB_USER}/result:${VERSION}", "./result")
                    docker.build("${DOCKERHUB_USER}/worker:${VERSION}", "./worker")
                }
            }
        }

        stage('Push to DockerHub') {
            steps {
                script {
                    docker.withRegistry('', DOCKERHUB_CREDS) {
                        docker.image("${DOCKERHUB_USER}/vote:${VERSION}").push()
                        docker.image("${DOCKERHUB_USER}/result:${VERSION}").push()
                        docker.image("${DOCKERHUB_USER}/worker:${VERSION}").push()
                    }
                }
            }
        }

        stage('Update GitOps repo') {
            steps {
                sshagent([GITOPS_CREDS]) {
                    sh """
                        # Clone the GitOps repo into a temp directory
                        rm -rf gitops-tmp
                        git clone ${GITOPS_REPO} gitops-tmp
                        cd gitops-tmp

                        # Bump image tags — replaces everything after the colon with the new version
                        sed -i 's|image: ${DOCKERHUB_USER}/vote:.*|image: ${DOCKERHUB_USER}/vote:${VERSION}|'     vote-deployment.yaml
                        sed -i 's|image: ${DOCKERHUB_USER}/result:.*|image: ${DOCKERHUB_USER}/result:${VERSION}|' result-deployment.yaml
                        sed -i 's|image: ${DOCKERHUB_USER}/worker:.*|image: ${DOCKERHUB_USER}/worker:${VERSION}|' worker-deployment.yaml

                        # Commit and push
                        git config user.email "jenkins@devops-lab"
                        git config user.name "Jenkins"
                        git add vote-deployment.yaml result-deployment.yaml worker-deployment.yaml
                        git commit -m "ci: bump image tags to ${VERSION} [build #${BUILD_NUMBER}]"
                        git push origin main
                    """
                }
            }
        }
    }

    post {
        always {
            sh "rm -rf gitops-tmp"
            cleanWs()
        }
    }
}
