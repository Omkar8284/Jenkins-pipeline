pipeline {
    agent any
    environment {
        DEPLOY_USER   = "omkar"
        NODES         = "192.168.152.129"       // single web node in your current setup
        RELEASE_DIR   = "/var/www/prod/releases"
        CURRENT_DIR   = "/var/www/prod/current"
        SSH_KEY       = "jenkins-ssh"           // Jenkins SSH credential ID you will create
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
                  TIMESTAMP=$(date +%Y%m%d%H%M%S)
                  mkdir -p build
                  cp -r index.html build/
                  cd build
                  tar -czf site-$TIMESTAMP.tgz *
                  echo $TIMESTAMP > ../build/release_id
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
                        scp build/site-$RELEASE_ID.tgz ${DEPLOY_USER}@$node:${RELEASE_DIR}/
                        ssh ${DEPLOY_USER}@$node "mkdir -p ${RELEASE_DIR}/$RELEASE_ID && tar -xzf ${RELEASE_DIR}/site-$RELEASE_ID.tgz -C ${RELEASE_DIR}/$RELEASE_ID && rm -f ${CURRENT_DIR} && ln -s ${RELEASE_DIR}/$RELEASE_ID ${CURRENT_DIR} && sudo systemctl reload nginx"
                      done
                    '''
                }
            }
        }
        stage('Healthcheck') {
            steps {
                sh 'for node in ${NODES}; do curl -sSf http://$node:8080/ || (echo "healthcheck failed for $node" && exit 1); done'
            }
        }
    }
    post {
        success { echo "Deployment successful" }
        failure { echo "Pipeline failed — inspect console" }
    }
}
