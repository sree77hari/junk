def mvnCmd = "mvn -s configuration/settings.xml"
def templatePath = "cicd/template-dev.json"

def quayRegistryHostname = "quay.io/srereddy/sample"
def quayRegistryOrgName = "srereddy/sample"
def clusterApiUrl= "https://api.cluster-crxbz.sandbox1198.opentlc.com:6443"
def ocpRegistryUrl ="default-route-openshift-image-registry.apps.cluster-crxbz.sandbox1198.opentlc.com"

pipeline {


    agent { label 'customagent' }

    environment {

        PROJECT = "fuse-dev"
        QAPROJECT = "fuse-qa"
        NAME = "getflightavailableinfo"
        ENV = "dev"
        QUAY_REPO_NAME="srereddy/sample"
        QUAY_URL="https://default-route-openshift-image-registry.apps.cluster-crxbz.sandbox1198.opentlc.com"
    }

    stages {

        stage('Checkout') {

            steps {
                checkout scm

            }

        } //End of Stage Checkout


        stage('Code  Compile') {

            steps {
                sh "${mvnCmd} clean package -DskipTests=true -Dversion=${env.BUILD_NUMBER}"
            }
        } //End of Stage Code Compile


        stage ('Deploy Template') {

                      steps{
                             script{
                                        try {
                                    openshift.withCluster() {
                                        openshift.withProject(env.PROJECT) {
                                                echo "Using project: ${openshift.project()}"

                                                def templateSelector = openshift.selector( "template", "${NAME}")


                                                if(!openshift.selector("dc", [ template : "${NAME}"]).exists() || !openshift.selector("svc", [ template : "${NAME}"]).exists() ||
                                                 !openshift.selector("route", [ template : "${NAME}"]).exists())
                                                {

                                                echo "In Deploy Template Stage and Template Modification Required"

                                                if(openshift.selector("dc", [ template : "${NAME}"]).exists()){
                                                openshift.selector("dc", "${NAME}").delete();
                                                }
                                                if(openshift.selector("svc", [ template : "${NAME}"]).exists()){
                                                openshift.selector("svc", "${NAME}").delete();
                                                }
                                                if(openshift.selector("route", [ template : "${NAME}"]).exists()){
                                                openshift.selector("route", "${NAME}").delete();
                                                }
                                                if(openshift.selector("bc", [ template : "${NAME}"]).exists()){
                                                openshift.selector("bc", "${NAME}").delete();
                                                }
                                                if(openshift.selector("is", [ template : "${NAME}"]).exists()){
                                                openshift.selector("is", "${NAME}").delete();
                                                }


                                                openshift.newApp(templatePath, "-p PROJECT=${env.PROJECT} -p NAME=${env.NAME} -p ENV=${env.ENV}")

                                                }
                                                 else {
                                                        echo "Using project: ${openshift.project()} , template already exists"
                                                }

                                                }
                                                }
                                                }

                                         catch ( e ) {
                                                echo e.getMessage()
                                                error "Deploy Template not successful."
                                             }
                                 }
                                    }

                         } // End of Stage Deploy Template

        stage('Image build') {

                                  steps{
                                          script{
                                            try {
                                            timeout(time: 5, unit: 'MINUTES') {
                                                                openshift.withCluster() {
                                                                openshift.withProject(env.PROJECT) {
                                                                echo "Using project in Image Build ${openshift.project()}  ${NAME}-${env.BUILD_NUMBER}.jar"
                                                                def build = openshift.selector("bc", "${NAME}").startBuild("--from-file=target/${NAME}-${env.BUILD_NUMBER}.jar", "--wait=true")
                                                                build.untilEach {
                                                                        echo "Using project in Image Build ${build}"
                                                                        return it.object().status.phase == "Complete"
                                                                }
                                                     }
                                                 }
                                             }
                                             echo "STAGE Image Build Template Finished"
                                     }
                                           catch ( e ) {
                                                echo e.getMessage()
                                                error "Build not successful."
                                       }
                                     }
                                  }
                               } // End of Stage Image build


           stage('Push Image to QUAY') {
                 steps {
                      retry (count : 3) {
                    script {
                         withCredentials([usernamePassword(credentialsId: 'quayPass', passwordVariable: 'QUAY_REGISTRY_USER_PASS', usernameVariable: 'QUAY_REGISTRY_USER_NAME'),
                         usernamePassword(credentialsId: 'ocpClusterCreds', passwordVariable: 'CLUSTER_USER_PASS', usernameVariable: 'CLUSTER_USER_NAME')])
                         {
                           sh "oc login -u  ${CLUSTER_USER_NAME} -p \"${CLUSTER_USER_PASS}\" --insecure-skip-tls-verify=true ${clusterApiUrl} "
                           def temptoken = sh(script: 'echo -n ${CLUSTER_USER_NAME}:`oc whoami -t`  | base64 ', returnStdout: true).trim()
                           echo "temptoken is '${temptoken}'"
                           registryEncodedToken  = temptoken.replaceAll("\n", "")

                           def quaytoken = sh(script: 'echo -n ${QUAY_REGISTRY_USER_NAME}:${QUAY_REGISTRY_USER_PASS}  | base64 ', returnStdout: true).trim()

                           sh """
                           echo "registryEncodedToken is '${registryEncodedToken}'"
                           echo "quaytoken is '${quaytoken}'"
                           cp /home/jenkins/docker/config.json /tmp/config.json
                           cat  /tmp/config.json
                           sed -i "s/SourceRegistryPass/$quaytoken/g" /tmp/config.json
                           sed -i "s/SourceRegistry/$quayRegistryHostname/g"  /tmp/config.json
                           sed -i "s/DestRegistryPass/$registryEncodedToken/g"  /tmp/config.json
                           sed -i "s/DestRegistry/$ocpRegistryUrl/g"  /tmp/config.json

                           cat  /tmp/config.json

                           oc image mirror -a  /tmp/config.json  --insecure=true ${ocpRegistryUrl}/${PROJECT}/${NAME}:latest  ${quayRegistryHostname}/${quayRegistryOrgName}:${env.BUILD_NUMBER}

                           """
                    }
                    }
      }
               }
         } // STAGE Push Image to QUAY END


           stage('Vulnerability Scan') {

            steps {
                    script {
                     withCredentials([
                     usernamePassword(credentialsId: 'quayapptoken', passwordVariable: 'QUAY_REGISTRY_USER_TOKEN', usernameVariable: 'username')]){
                        openshift.withCluster() {
                        openshift.withProject(env.DEV_NAMESPACE) {
                            sh """
                                echo "Setting the QUAY_REGISTRY_USER_TOKEN "
                                export tag=${env.BUILD_NUMBER}
                                export QUAY_API_TOKEN=${QUAY_REGISTRY_USER_TOKEN}
                                export IMAGE_ID_OUTPUT_FILE=/tmp/image-id.txt
                                export VULNERABILITY_OUTPUT_FILE=/tmp/Vulnerability.txt
                                sleep 30s
                                /vulnerability_scan.sh 2> /dev/null || exit 0

                            """
                        }
                        }
                        }
                    }
            }
        }

        stage('Deployment Confirmation') {

            steps {
                    script {
                        input message: 'Do you want to Deploy the application ${NAME}?'
                    }
            }
        }





            stage('Deployment in Dev Namespace') {
                                             steps{
                                                  script{
                                                         try {
                                                                timeout(time: 5, unit: 'MINUTES') {
                                                                 openshift.withCluster() {
                                                                 openshift.withProject(env.PROJECT) {

                                                                        echo "Using project: ${openshift.project()}"
                                                                        if (openshift.selector("cm", "${NAME}-${ENV}").exists()){
                                                                            openshift.selector("cm", "${NAME}-${ENV}").delete();
                                                                        }
                                                                        openshift.create("cm","${NAME}-${ENV}","--from-file=cicd/${NAME}-${ENV}.properties")

                                                                        def deploy= openshift.selector("dc", "${NAME}")
                                                                        deploy.rollout().latest();
                                                                  }
                                                        }
                                                     }
                                                               }
                                                               catch ( e ) {
                                                                error "Deployment not successful."
                                                             }
                                                  }
                                              }
                                         } // End of Deployment


             stage('Image Tag') {
                   steps {
                                script {
                                      openshift.withCluster() {
                                      openshift.tag("${PROJECT}/${NAME}:latest", "${PROJECT}/${NAME}:${env.BUILD_NUMBER}")
                                      openshift.tag("${PROJECT}/${NAME}:${env.BUILD_NUMBER}", "${QAPROJECT}/${NAME}:latest")
                             }

                              echo "STAGE Image Tag  Finished"
                        }
                    }
                 }




    } //End of Stages

} //End of Pipeline
