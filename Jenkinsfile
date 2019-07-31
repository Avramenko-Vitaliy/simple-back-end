node {
    timeout(20){
        try {
            deleteDir() // Clean the workspace
            notifyBuild()

            environment{
               HASH_COMMIT = ''
            }
            stage('Checkout') {
                git branch: 'master',
                    url: 'https://github.com/Avramenko-Vitaliy/simple-back-end'

                env.HASH_COMMIT = sh(script: 'git rev-parse HEAD', returnStdout: true).trim()
                echo env.HASH_COMMIT
            }

            stage('Run tests and build docker') {
                sh 'mvn clean install -Pdocker -Ddocker.image.name=130114285352.dkr.ecr.us-east-1.amazonaws.com/simple-back -Ddocker.image.tag=${HASH_COMMIT}'
            }

            stage('Push simple-back-end image') {
                sh 'rm  ~/.dockercfg || true'
                sh 'rm ~/.docker/config.json || true'

                //configure registry
                docker.withRegistry('https://130114285352.dkr.ecr.us-east-1.amazonaws.com', 'ecr:us-east-1:ead6e682-bbc4-4b71-8863-af5167d782a4') {
                    docker.image('130114285352.dkr.ecr.us-east-1.amazonaws.com/simple-back:${HASH_COMMIT}').push()
                }
            }

            stage('Redeploy service') {
                git branch: 'capstone',
                       url: 'https://github.com/Avramenko-Vitaliy/itea-devops'

                withCredentials([[$class: 'AmazonWebServicesCredentialsBinding', credentialsId: 'ead6e682-bbc4-4b71-8863-af5167d782a4', variable: 'AWS_ACCESS_KEY_ID']]) {
                    sh 'echo this is ${env.AWS_ACCESS_KEY_ID}'
                    sh 'echo this is ${env.AWS_SECRET_ACCESS_KEY}'

                    sh 'cd capstone/service'
                    sh 'terraform init -var="ecr_image_tag=${HASH_COMMIT}" -var="aws_access_key_id=${env.AWS_ACCESS_KEY_ID}" -var="aws_secret_access_key=${env.AWS_SECRET_ACCESS_KEY}"'
                    sh 'terraform apply -auto-approve -target=aws_ecs_service.ecs-service -var="ecr_image_tag=${HASH_COMMIT}" -var="aws_access_key_id=${env.AWS_ACCESS_KEY_ID}" -var="aws_secret_access_key=${env.AWS_SECRET_ACCESS_KEY}"'
                }
            }

            stage('Clear images') {
                sh 'docker rmi $(docker images -q)'
            }

        } catch (e) {
            sh 'exit 1'
            currentBuild.result = "FAILED"
            throw e
        } finally {
            notifyBuild(currentBuild.result)
        }
    }
}
def notifyBuild(String buildStatus = 'STARTED') {
  // build status of null means successful
  buildStatus =  buildStatus ?: 'SUCCESS'

  // Default values
  def colorName = 'RED'
  def colorCode = '#FF0000'
  def subject = "${buildStatus}: Job '${env.JOB_NAME} [${env.BUILD_NUMBER}]'"
  def summary = "${subject} (${env.BUILD_URL})"

  // Override default values based on build status
  if (buildStatus == 'STARTED') {
    color = 'YELLOW'
    colorCode = '#FFFF00'
  } else if (buildStatus == 'SUCCESS') {
    color = 'GREEN'
    colorCode = '#00FF00'
  } else {
    color = 'RED'
    colorCode = '#FF0000'
    summary = "@nick @channel ${subject} (${env.BUILD_URL})"
  }
}
