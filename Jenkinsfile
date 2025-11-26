pipeline {
    agent any

    environment {
        DEPLOY_USER   = "omkar"
        NODES         = "192.168.152.129"
        RELEASE_DIR   = "/var/www/prod/releases"
        CURRENT_DIR   = "/var/www/prod/current"
        SSH_KEY       = "jenkins-ssh"
    }

    stages {

        stage('Checkout') {
            steps { checkout scm }
        }

        stage('Validate') {
            steps {
                sh 'test -f index.html || (echo "index.html missing" && exit 1)'
            }
        }

        stage('Package') {
            steps {
                sh '''
                  rm -rf build
                  TIMESTAMP=$(date +%Y%m%d%H%M%S)
                  mkdir -p build
                  cp index.html build/
                  cd build
                  tar -czf site-$TIMESTAMP.tgz index.html
                  echo $TIMESTAMP > release_id
                '''
            }
        }

        stage('Deploy') {
            steps {
                sshagent([SSH_KEY]) {
                    sh '''
                      RELEASE_ID=$(cat build/release_id)
                      for node in ${NODES}; do
                        echo "Deploying to $node ..."
                        scp -o StrictHostKeyChecking=no build/site-$RELEASE_ID.tgz ${DEPLOY_USER}@$node:${RELEASE_DIR}/
                        ssh -o StrictHostKeyChecking=no ${DEPLOY_USER}@$node "
                          mkdir -p ${RELEASE_DIR}/$RELEASE_ID && \
                          tar -xzf ${RELEASE_DIR}/site-$RELEASE_ID.tgz -C ${RELEASE_DIR}/$RELEASE_ID && \
                          sudo rm -rf ${CURRENT_DIR} && \
                          sudo ln -s ${RELEASE_DIR}/$RELEASE_ID ${CURRENT_DIR} && \
                          sudo systemctl reload nginx
                        "
                      done
                    '''
                }
            }
        }

        stage('Healthcheck') {
            steps {
                sh '''
                  for node in ${NODES}; do
                    curl -sSf http://$node:8080/ \
                    || (echo "healthcheck failed for $node" && exit 1);
                  done
                '''
            }
        }
    }

    post {
        success {
            echo "🎉 Deployment Successful!"
        }
        failure {
            echo "❌ Pipeline failed — check console"
        }
    }
}
