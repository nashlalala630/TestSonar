#!groovy

group = "logistics"
product = "parcel-packing"
application = "order-fc-provider-api"

gitUrl = 'ssh://git@stash.zooplus.de:22/pp/order-fc-provider-api.git'
gitCredentials = 'bb1e890e-7bdd-4c23-9a63-3b78b8cb616b'

marathonDevUrl = 'http://marathon.dev.zooplus.net:8080'
marathonProdUrl = 'http://marathon.prod.zooplus.net:8080'

MARATHON_JSON_TEMPLATE_PATH = 'marathon/marathon.json.tpl'
formattedBranchName = env.BRANCH_NAME.replaceAll('/', '-').toLowerCase()
success = true

try {
//    node('java8-dind-maven3') {
    node {
//        timestamps {
//            dir("$env.BRANCH_NAME/$env.BUILD_NUMBER") {

//                stage('Clone') {
//                    checkout scm
//                    stash name: 'marathon_file', includes: 'marathon/*.tpl'
//                    env.DOCKER_IMAGE = getImageVersion(formattedBranchName) // used in marathon.json.tpl
//                    lastCommitAuthorEmail = getLastCommitAuthorEmail()
//                }

                stage('Compile') {
                    sh 'mvn compile'
                    env.JAR_VERSION = readMavenPom().version
                }

                stage('Test') {
                    sh 'mvn test'
                }

                stage('Build') {
                    sh 'mvn -DskipTests package'
                    sh 'cp api/target/*.jar docker/'
                }

//                stage('Dockerize') {
//                    dir('docker') {
//                        sh "docker build --build-arg version=$env.JAR_VERSION -t $env.DOCKER_IMAGE ."
//                        sh "docker push $env.DOCKER_IMAGE"
//                    }
//                }
//            }
//        }
    }

//    env.MARATHON_APP_ID = "$group/$product/$application"    // used in marathon.json.tpl
//    env.APPLICATION_NAME = application
//    env.MARATHON_APP_LABEL_FRONTENDS = application
//    env.LOG_FORMAT = "json"
//    env.INSTANCES = 1
//
//    deployToDev = onDevelopBranch()
//    deployToProd = false
//
//    if (onBranchDeployableToProduction()) {
//        stage('Should it run on PRODUCTION') {
//            try {
//                timeout(time: 2, unit: 'HOURS') {
//                    input message: "Do you want to deploy to PRODUCTION?"
//                }
//                deployToProd = true
//            } catch (ignored) {
//                echo "Timeout of waiting for manual deploy to PRODUCTION"
//            }
//        }
//    }
//
//    if (deployToDev || deployToProd) {
//        node('java8-dind-maven3') {
//            stage('Deploy on ' + (deployToDev ? 'DEVELOPMENT' : 'PRODUCTION')) {
//                timestamps {
//                    dir("$formattedBranchName/$env.BUILD_NUMBER") {
//                        unstash 'marathon_file'
//
//                        if (deployToProd) {
//                            deployOnMarathon(marathonProdUrl, ["ZOOPLUS_ENV=prod", "MACHINE_ENV=ops85"])
//                        } else {
//                            deployOnMarathon(marathonDevUrl, ["ZOOPLUS_ENV=test"])
//                        }
//                    }
//                }
//            }
//        }
//    }
} catch (ex) {
    success = false
    if (onDevelopBranch())
        notifyAboutFail()
    throw ex
} finally {
    if (onDevelopBranch() && success && currentBuild.previousBuild.result == 'FAILURE')
        notifyAboutRecover()
}

def notifyAboutFail() {
    mail(
            subject: "Logistics Order FC Provider API build on branch $env.BRANCH_NAME failed",
            body: "Check console output at ${env.BUILD_URL} to view the results.",
            to: lastCommitAuthorEmail
    )
}

def notifyAboutRecover() {
    mail(
            subject: "Logistics Order FC Provider API build on branch $env.BRANCH_NAME is back to normal",
            body: 'Build on branch $env.BRANCH_NAME is back to normal. Congratulations! $env.BUILD_URL',
            to: lastCommitAuthorEmail
    )
}

def getLastCommitAuthorEmail() {
    sh(returnStdout: true, script: 'git show -s --format="%ae"').trim()
}

def getImageVersion(branchName) {
    if (onBranchDeployableToProduction()) {
        version = "v" + readMavenPom().version + "-" + getCurrentTimestamp() + "." + getCommitHash()
        "repo.zooplus.de/$group/$product/$application:$version"
    } else {
        "repo.zooplus.de/$group/$product/$application/$branchName"
    }
}

def getCommitHash() {
    sh(returnStdout: true, script: 'git rev-parse --short HEAD').trim()
}

def getCurrentTimestamp() {
    sh(returnStdout: true, script: 'date +%Y%m%d-%H%M%S').trim()
}

def onBranchDeployableToProduction() {
    env.BRANCH_NAME.startsWith('release/')
}

def onDevelopBranch() {
    'develop' == env.BRANCH_NAME
}

def deployOnMarathon(deployUrl, environment) {
    withEnv(environment) {
        echo 'Triggering update...'
        sh "tpl -t ${MARATHON_JSON_TEMPLATE_PATH} | curl -sfL -H \"Content-Type: application/json\" -X PUT $deployUrl/v2/apps/${env.MARATHON_APP_ID}?force=true -d @-"
        sleep(time: 2, unit: 'SECONDS')
        echo 'Triggering restart...'
        // in case an update does not trigger a deployment due to no changes to rendered .json config
        sh "curl -sfL -X POST $deployUrl/v2/apps/${env.MARATHON_APP_ID}/restart | jq ."

        timeout(time: 3, unit: 'MINUTES') {
            waitUntil {
                sh "curl -sfL $deployUrl/v2/deployments | jq . | tee .deploymentsAll"
                sh "jq -r '.[].affectedApps[]' .deploymentsAll | grep '^${env.MARATHON_APP_ID}' | wc -l > .deploymentsAffected"
                def deploymentsAffected = readFile('.deploymentsAffected').trim()
                if (deploymentsAffected == "0") {
                    echo 'Successfuly deployed!'
                    return true
                } else {
                    echo 'Waiting for deployment to finish...'
                    sleep(time: 10, unit: 'SECONDS')
                    return false
                }
            }
        }
    }
}

