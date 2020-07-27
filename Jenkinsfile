// ================================================================
// Name: Development pipeline
//
// Description: This pipeline builds artifacts, tests artifacts and
//              deploys to development environment
//
// Usage: Pipeline needs to be configured with service specific
//        information
// ================================================================


// ================================================================
// Define a pod template to use for the Jenkins slave pod
// jenkins-slave-maven , skopeo
// ================================================================

podTemplate(
        label: "jenkins-slave-maven",
        cloud: "openshift",
        inheritFrom: "maven",
        containers: [
                containerTemplate(
                        name: "jnlp",
                        image: "docker-registry.default.svc:5000/cicd/jenkins-slave-maven",
                        resourceRequestMemory: "1Gi",
                        resourceLimitMemory: "1Gi",
                )
        ]
)



        // Pipeline body

        {
          node('jenkins-slave-maven') {

            // Constants, please configure for to your service
            def SERVICE_NAME = "springboot-actuator"  // Name of your service example: nl-appointment-dom
            def BUILD_IMAGE = "openjdk18-openshift" // Docker runtime image
            def DEV_PROJECT = "pringboot-actuator" // Development namespace example: nl-customer-dev
            def CREATE_ROUTE = "true" // Should a route automatically be created

            // Resources to be assigned to your deployment
            def CPU_REQUESTS = "200m"
            def CPU_LIMITS = "300m"
            def MEM_REQUESTS = "512Mi"
            def MEM_LIMITS = "1Gi"


            // Checkout Source Code
            stage('Checkout Source') {
              //source is in same repo as Jenkinsfile
              checkout scm
            }

            // Define Maven Command. Make sure it points to the correct settings for our Nexus installation
            // The file maven-settings.xml needs to be in the Source Code repository.
           // def mvnCmd = "mvn -s maven-settings.xml"
            def mvnCmd = "mvn "
            echo "mvnCmd: ${mvnCmd}"

            dir('.') {
              // ================================================================================================
              // Calculate versions and tags to use for build artifacts and containers
              // ================================================================================================

              // The following variables need to be defined at the top level
              // and not inside the scope of a stage - otherwise they would not be accessible from other stages.

              // Extract version and other properties from the pom.xml
              def groupId    = getGroupIdFromPom("pom.xml")
              echo "groupId: ${groupId}"
              def artifactId = getArtifactIdFromPom("pom.xml")
              echo "artifactId: ${artifactId}"
              def version    = getVersionFromPom("pom.xml")
              def pomVersion    = "${version}"
              echo "pomVersion: ${pomVersion}"

              // Get git commit hash to link to version
              GIT_COMMIT_HASH = sh (script: "git log -n 1 --pretty=format:'%H'", returnStdout: true)
              GIT_COMMIT_HASH = GIT_COMMIT_HASH.toString().substring(0,7)
              def commitId = "${GIT_COMMIT_HASH}"
              echo "commitId: ${commitId}"

              // Get version
              if (getVersionFromPom("pom.xml").toString().toUpperCase().contains("SNAPSHOT")) {
                version = getVersionFromPom("pom.xml").toString().toUpperCase().replace("-SNAPSHOT", "")
              } else {
                //If the condition is false print the following statement
                version = getVersionFromPom("pom.xml").toString();
                println("versionInfo without snapshot extension :" +version);
              }
              echo "version:  ${version}"


              def devTag  = "${version}-${commitId}"

              // Set the tag for the production image: version -- to be discussed
              def prodTag = "${version}"

              def path = getGroupIdFromPom("pom.xml").toString().replace(".", "/")
              echo "package path: ${path}"

              // ===================================================================================================



              // Using Maven build the war file
              // Do not run tests in this step
              stage('Build Application Binary') {
                echo "Building version ${prodTag}"
                sh "${mvnCmd} clean package -DskipTests=true"
              }

              // Using Maven run the unit tests
              stage('Unit Tests') {
                echo "Running Unit Tests"
                sh "${mvnCmd} test -DskipTests=false"
              }

              // Using Maven call SonarQube for Code Analysis and quality
              stage('Code Analysis') {
                echo "Running Code Analysis"
                // sh "${mvnCmd} sonar:sonar -Dsonar.projectName=${SERVICE_NAME} -Dsonar.projectVersion=${prodTag}"
              }

              // Publish the built war file to Nexus
              stage('Publish to Nexus') {
                echo "Publish to Nexus"
                // sh "${mvnCmd} deploy:deploy-file -DgroupId=${groupId} -DartifactId=${artifactId} -Dversion=${prodTag} -Dpackaging=jar -DrepositoryId=nexus -Durl=http://nexus3.cicd.svc.cluster.local:8081/repository/releases -Dfile=target/${artifactId}-${pomVersion}.jar -DskipTests=true -DpomFile=pom.xml"
              }

              // Create or replace Image builder artifacts
              stage('Create Image Builder') {
                echo "Creating Image Builder"
                sh "if oc get bc ${SERVICE_NAME} --namespace=cicd; \
                then echo \"exist\"; \
                else oc new-build --binary=true --name=${SERVICE_NAME} ${BUILD_IMAGE} --labels=app=${SERVICE_NAME} -n cicd;fi"
              }

              // Build the OpenShift Image in OpenShift and tag it.
              stage('Build and Tag OpenShift Image') {
                echo "Building OpenShift container image ${SERVICE_NAME}:${prodTag}"

                // Start Binary Build in OpenShift CICD cluster using the file we just published
                sh "oc start-build ${SERVICE_NAME} --follow --from-file=target/${artifactId}-${pomVersion}.jar -n cicd"
                echo "oc start build complete."

                // Tag the latest image to prodTag
                sh "oc tag ${SERVICE_NAME}:latest ${SERVICE_NAME}:${prodTag} --alias=false -n cicd"
              }

              stage('Copy Image to Application Project'){
                 sh "oc tag ${SERVICE_NAME}:latest ${DEV_PROJECT}/${SERVICE_NAME}:${prodTag}"

                 }
              }

              // Deploy the built image to the Development Environment.
              stage('Configure Deployment') {
                echo "Deploying container image to Development Project"
                openshift.withCluster(){
                  echo "Using openshift cluster ${openshift.cluster()}"

                  // Switch to target project on remote cluster
                  openshift.withProject( "${DEV_PROJECT}" ) {
                    echo "Using project: ${openshift.project()}"

                    //Create application if it does not exist
                    def deploymentSelector = openshift.selector( "dc", "${SERVICE_NAME}")
                    def deploymentExists = deploymentSelector.exists()
                    if (!deploymentExists) {
                      echo "Deployment ${SERVICE_NAME} does not exist"
                      openshift.newApp("${DEV_PROJECT}/${SERVICE_NAME}:0.0.0", "--name=${SERVICE_NAME}", "--allow-missing-imagestream-tags=true","--namespace=${DEV_PROJECT}")
                    }
                    echo "Deployment ${SERVICE_NAME} exists"

                    // Disable triggers, we want to control manually
                    openshift.raw("set","triggers","dc/${SERVICE_NAME}","--remove-all","--namespace","${DEV_PROJECT}")

                    // Set resource requirements
                    try {
                      def result = openshift.raw("set","resources","dc/${SERVICE_NAME}","--limits=cpu=${CPU_LIMITS},memory=${MEM_LIMITS}","--requests=cpu=${CPU_REQUESTS},memory=${MEM_REQUESTS}","--namespace=${DEV_PROJECT}")
                    } catch(e) {
                      echo e.getMessage()
                      if(!e.getMessage().contains("info:")){
                        throw e
                      }
                    }

                    //Set probes
                    openshift.raw("set","probe","dc/${SERVICE_NAME}","--namespace=${DEV_PROJECT}","--liveness","--failure-threshold","3","--initial-delay-seconds","200","--get-url=http://:9090/actuator/health")
                    openshift.raw("set","probe","dc/${SERVICE_NAME}","--namespace=${DEV_PROJECT}","--readiness","--failure-threshold 3","--initial-delay-seconds","200","--get-url=http://:9090/actuator/health")

                    try {
                      //Create service
                      openshift.raw("expose","dc ${SERVICE_NAME}","--name ${SERVICE_NAME}","--port 8080,9090", "--protocol=TCP","-n ${DEV_PROJECT}")
                      def service = openshift.selector( "svc", "${SERVICE_NAME}").object()
                      service.spec.ports[0].name='8080-tcp'
                      service.spec.ports[1].name='9090-tcp'
                      openshift.apply(service)

                    } catch(e) {
                      echo e.getMessage()
                      if(!e.getMessage().contains("AlreadyExists")){
                        throw e
                      }
                    }


                    //Update the Image on the Development Deployment Config
                    openshift.raw("set","image dc/${SERVICE_NAME}","${SERVICE_NAME}=${SERVICE_NAME}:${prodTag}","--source=imagestreamtag",
                            "-n ${DEV_PROJECT}")

                  }
                }
              }

              // Re-Create configMap.
              stage('Create configMap') {
                openshift.withCluster(){
                  openshift.withProject( "${DEV_PROJECT}" ) {
                    def configmapSelector = openshift.selector( "cm", "${SERVICE_NAME}")
                    def configmapExists = configmapSelector.exists()

                    if (configmapExists) {
                      echo "Configmap ${SERVICE_NAME} exists"
                      configmapSelector.delete()
                    }

                    openshift.create("configmap ${SERVICE_NAME}", "--from-file=./src/main/resources/application.yml")

                    echo "Label ConfigMap"

                    def cm = openshift.selector( "cm", "${SERVICE_NAME}").object()
                    cm.metadata['labels.app']='${SERVICE_NAME}' // Adjust the model
                    openshift.apply(cm)

                  }
                }
              }



              // Load configMap as a volume
              stage('Load configMap as a volume') {
                openshift.withCluster('non-prod'){
                  openshift.withProject( "${DEV_PROJECT}" ) {

                    openshift.raw("set","volumes dc/${SERVICE_NAME}","--add --overwrite=true \
                 --name=config-volume --mount-path=/deployments/config -t configmap \
                 --configmap-name=${SERVICE_NAME}","-n ${DEV_PROJECT}")
                  }
                }
              }


              // Load logging - configMap as a volume
              stage('Load logging configMap as a volume') {
                openshift.withCluster('non-prod'){
                  openshift.withProject( "${DEV_PROJECT}" ) {

                    openshift.raw("set","volumes dc/${SERVICE_NAME}","--add --overwrite=true \
                 --name=logconfig-volume --mount-path=/deployments/logconfig -t configmap \
                 --configmap-name=logback-mapping","-n ${DEV_PROJECT}")
                  }
                }
              }

              // // Set Spring Profile
              stage('Set Environment variables') {
                openshift.withCluster('non-prod'){
                  openshift.withProject( "${DEV_PROJECT}" ) {

                    openshift.raw("set","env dc/${SERVICE_NAME} JAVA_OPTIONS='-Dspring.profiles.active=openshift-dev -Xverify:none -Xms400m -Xmx800m'", "-n ${DEV_PROJECT}")
                    openshift.raw("set","env dc/${SERVICE_NAME} UNSET_PROXY='true'", "-n ${DEV_PROJECT}")
                  }
                }
              }

              stage("Rollout to Developmemt"){
                openshift.withCluster('non-prod'){
                  openshift.withProject( "${DEV_PROJECT}" ) {
                    def deployment = openshift.selector('dc', "${SERVICE_NAME}")
                    deployment.rollout().latest()

                    timeout(time:10, unit:'MINUTES') {
                      deployment.rollout().status()
                    }
                  }
                }
              }


            }
          }
        }

// Convenience Functions to read variables from the pom.xml
// Do not change anything below this line.
def getVersionFromPom(pom) {
  def matcher = readFile(pom) =~ '<version>(.+)</version>'
  matcher ? matcher[0][1] : null
}
def getGroupIdFromPom(pom) {
  def matcher = readFile(pom) =~ '<groupId>(.+)</groupId>'
  matcher ? matcher[0][1] : null
}
def getArtifactIdFromPom(pom) {
  def matcher = readFile(pom) =~ '<artifactId>(.+)</artifactId>'
  matcher ? matcher[0][1] : null
}