def ENV_NAME = getEnvName(env.BRANCH_NAME)
def CONTAINER_NAME = "calculator-"+ENV_NAME
def CONTAINER_TAG = getTag(env.BUILD_NUMBER, env.BRANCH_NAME)
def HTTP_PORT = getHTTPPort(env.BRANCH_NAME)
def EMAIL_RECIPIENTS = "scopetest24@gmail.com"

node {
    try {
        stage('Initialize') {
            def dockerHome = tool 'dockerlatest'
            def mavenHome = tool 'mavenlatest'
            env.PATH = "${dockerHome}/bin:${mavenHome}/bin:${env.PATH}"
        }

        stage('Checkout') {
            checkout scm
        }

        stage('Build with test') {
            sh "mvn clean install"
        }

        stage('Sonarqube Analysis') {
            withSonarQubeEnv('SonarQubeLocalServer') {
                sh "mvn sonar:sonar -Dintegration-tests.skip=true -Dmaven.test.failure.ignore=true"
            }

            // Timeout augmenté pour éviter le problème d'attente
            timeout(time: 10, unit: 'MINUTES') {
                def qg = waitForQualityGate()
                if (qg.status != 'OK') {
                    error "Pipeline aborted due to quality gate failure: ${qg.status}"
                }
            }
        }

        stage("Docker: Image Prune") {
            imagePrune(CONTAINER_NAME)
        }

        stage('Docker: Image Build') {
            imageBuild(CONTAINER_NAME, CONTAINER_TAG)
        }

        stage('Docker: Push to Registry') {
            withCredentials([usernamePassword(credentialsId: 'dockerhubcredentials', usernameVariable: 'USERNAME', passwordVariable: 'PASSWORD')]) {
                pushToImage(CONTAINER_NAME, CONTAINER_TAG, USERNAME, PASSWORD)
            }
        }

        stage('Docker: Run App') {
            withCredentials([usernamePassword(credentialsId: 'dockerhubcredentials', usernameVariable: 'USERNAME', passwordVariable: 'PASSWORD')]) {
                runApp(CONTAINER_NAME, CONTAINER_TAG, USERNAME, HTTP_PORT, ENV_NAME)
            }
        }

    } finally {
        deleteDir()
        sendEmail(EMAIL_RECIPIENTS)
    }
}

// ---------------- Docker functions ----------------
def imagePrune(containerName) {
    try {
        sh "docker image prune -f"
        sh "docker stop $containerName || true"
    } catch (ignored) {}
}

def imageBuild(containerName, tag) {
    sh "docker build -t $containerName:$tag -t $containerName --pull --no-cache ."
    echo "Image build complete"
}

def pushToImage(containerName, tag, dockerUser, dockerPassword) {
    sh "docker login -u $dockerUser -p $dockerPassword"
    sh "docker tag $containerName:$tag $dockerUser/$containerName:$tag"
    sh "docker push $dockerUser/$containerName:$tag"
    echo "Image push complete"
}

def runApp(containerName, tag, dockerHubUser, httpPort, envName) {
    sh "docker pull $dockerHubUser/$containerName:$tag"
    sh "docker run --rm --env SPRING_ACTIVE_PROFILES=$envName -d -p $httpPort:$httpPort --name $containerName $dockerHubUser/$containerName:$tag"
    echo "Application started on port: ${httpPort} (http)"
}

// ---------------- Utility functions ----------------
def sendEmail(recipients) {
    mail(
        to: recipients,
        subject: "Build ${env.BUILD_NUMBER} - ${currentBuild.currentResult} - (${currentBuild.fullDisplayName})",
        body: "Check console output at: ${env.BUILD_URL}/console\n"
    )
}

String getEnvName(String branchName) {
    if (branchName == 'main') return 'prod'
    if (branchName == 'develop') return 'uat'
    return 'dev'
}

String getHTTPPort(String branchName) {
    if (branchName == 'main') return '9003'
    if (branchName == 'develop') return '9002'
    return '9001'
}

String getTag(String buildNumber, String branchName) {
    if (branchName == 'main') return buildNumber + '-unstable'
    return buildNumber + '-stable'
}
