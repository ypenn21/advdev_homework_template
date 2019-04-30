// Set your project Prefix
def prefix      = "9187"
def GUID      = "9187"

// Set variable globally to be available in all stages
// Set Maven command to always include Nexus Settings
def mvnCmd      = "mvn -f ./openshift-tasks/pom.xml -s ./nexus_settings.xml"
// Set Development and Production Project Names
def devProject  = "9187-tasks-dev"
def prodProject = "9187-tasks-prod"
// Set the tag for the development image: version   build number
def devTag      = "0.0-0"
// Set the tag for the production image: version
def prodTag     = "0.0"
def destApp     = "tasks-green"
def activeApp   = ""

podTemplate(
  name: "jenkins-agent-appdev",
  label: "jenkins-agent-appdev",
  cloud: "openshift",
  containers: [
    containerTemplate(
      name: "jnlp",
      image: "docker-registry.default.svc:5000/${GUID}-jenkins/jenkins-agent-appdev:latest",
      resourceRequestMemory: "1Gi",
      resourceLimitMemory: "2Gi",
      resourceRequestCpu: "1",
      resourceLimitCpu: "2"
    )
  ]
) {
pipeline {
  agent {
    // Using the Jenkins Agent Pod that we defined earlier
    label "jenkins-agent-appdev"
  }
  
  stages {
      
    stage('Test skopeo') {
        steps {
            sh("skopeo --version")
            sh("oc whoami")
        }
    }
      
      // Run Integration Tests in the Development Environment.
    // Checkout Source Code and calculate Version Numbers and Tags
    stage('Checkout Source') {
        steps {
            // Replace xyz-gogs with the name of your Gogs project
            // Replace the credentials with your credentials.
            git credentialsId: 'github', url: 'https://github.com/ypenn21/advdev_homework_template'
            // or when using the Pipeline from the repo itself:
            // checkout scm

            script {
                def pom = readMavenPom file: './openshift-tasks/pom.xml'
                def version = pom.version

                // Set the tag for the development image: version   build number
                 devTag  = "${version}-"+currentBuild.number
                 // Set the tag for the production image: version
                prodTag = "${version}"

            }
        }
    }
    
        // some block
        // Using Maven build the war file
        // Do not run tests in this step
        stage('Build War File') {
            steps {
                echo "Building version ${devTag}"
                sh "${mvnCmd} clean package -DskipTests=true"
                // TBD

            }
        }
stage('Unit Tests and Code Analysis') {
  steps {
    parallel 'branch1': { 
            //  node('jenkins-agent-appdev') {  
                echo "Running Unit Tests"
                sh "${mvnCmd} test"
                // TBD
                // This next step is optional.
                // It displays the results of tests in the Jenkins Task Overview
                step([$class: 'JUnitResultArchiver', testResults: '**/target/surefire-reports/TEST-*.xml'])  
          
            // }
    }, 'branch2': {
            // node('jenkins-agent-appdev') {  
                echo "Running Code Analysis"
                sh "${mvnCmd} sonar:sonar -Dsonar.host.url=http://sonarqube-gpte-hw-cicd.apps.na311.openshift.opentlc.com -Dsonar.projectName=${JOB_BASE_NAME} -Dsonar.projectVersion=${devTag}"
          
            // }
        }
      
  }
}


        // Using Maven run the unit tests
    //     stage('Unit Tests') {
    //         steps {
    //             echo "Running Unit Tests"
    //             sh "${mvnCmd} test"
    //             // TBD
    //             // This next step is optional.
    //             // It displays the results of tests in the Jenkins Task Overview
    //             step([$class: 'JUnitResultArchiver', testResults: '**/target/surefire-reports/TEST-*.xml'])
    //          }
    //     }
        
    //         //Using Maven call SonarQube for Code Analysis
    // stage('Code Analysis') {
    //   steps {
    //     script {
    //         echo "Running Code Analysis"
    //         sh "${mvnCmd} sonar:sonar -Dsonar.host.url=http://sonarqube-gpte-hw-cicd.apps.na311.openshift.opentlc.com -Dsonar.projectName=${JOB_BASE_NAME} -Dsonar.projectVersion=${devTag}"
    //     }
    //   }
    // }
    
        // Publish the built war file to Nexus
        stage('Publish to Nexus') {
            steps {
                echo "Publish to Nexus"
                sh "${mvnCmd} deploy -DskipTests=true -DaltDeploymentRepository=nexus::default::http://nexus3-gpte-hw-cicd.apps.na311.openshift.opentlc.com/repository/releases"

            }
        }
        // Build the OpenShift Image in OpenShift and tag it.
    stage('Build and Tag OpenShift Image') {
      steps {
        echo "Building OpenShift container image tasks:${devTag}"
                // TBD: Tag the image using the devTag.
        script {
            openshift.withCluster() {
                openshift.withProject("${devProject}") {
                    openshift.selector("bc", "tasks").startBuild("--from-file=./openshift-tasks/target/openshift-tasks.war", "--wait=true")

                    // OR use the file you just published into Nexus:
                    // "--from-file=http://nexus3.${prefix}-nexus.svc.cluster.local:8081/repository/releases/org/jboss/quickstarts/eap/tasks/${version}/tasks-${version}.war"
                    openshift.tag("tasks:latest", "tasks:${devTag}")
                }
            }
        }
        // TBD: Start binary build in OpenShift using the file we just published.
        // Either use the file from your
        // workspace (filename to pass into the binary build
        // is openshift-tasks.war in the 'target' directory of
        // your current Jenkins workspace).
        // OR use the file you just published into Nexus:
        // "--from-file=http://nexus3.${prefix}-nexus.svc.cluster.local:8081/repository/releases/org/jboss/quickstarts/eap/tasks/${prodTag}/tasks-${prodTag}.war"


      }
    }
    
                // Deploy the built image to the Development Environment.
    stage('Deploy to Dev') {
      steps {
        echo "Deploy container image to Development Project"

        // TBD: Deploy the image
        // 1. Update the image on the dev deployment config
        // 2. Update the config maps with the potentially changed properties files
       // 3. Reeploy the dev deployment
       // 4. Wait until the deployment is running
       //    The following code will accomplish that by
       //    comparing the requested replicas
       //    (rc.spec.replicas) with the running replicas
       //    (rc.status.readyReplicas)
       //
        script {
            // Update the Image on the Development Deployment Config
            openshift.withCluster() {
            openshift.withProject("${devProject}") {
                openshift.set("image", "dc/tasks", "tasks=docker-registry.default.svc:5000/${devProject}/tasks:${devTag}")

            // For OpenShift 3 use this:
            // openshift.set("image", "dc/tasks", "tasks=docker-registry.default.svc:5000/${devProject}/tasks:${devTag}")

            // Update the Config Map which contains the users for the Tasks application
            // (just in case the properties files changed in the latest commit)
            openshift.selector('configmap', 'tasks-config').delete()
            def configmap = openshift.create('configmap', 'tasks-config', '--from-file=./openshift-tasks/configuration/application-users.properties', '--from-file=./openshift-tasks/configuration/application-roles.properties' )

            // Deploy the development application.
            openshift.selector("dc", "tasks").rollout().latest();

            // Wait for application to be deployed
            def dc = openshift.selector("dc", "tasks").object()
            def dc_version = dc.status.latestVersion
            def rc = openshift.selector("rc", "tasks-${dc_version}").object()

            echo "Waiting for ReplicationController tasks-${dc_version} to be ready"
            def loopNum = 0
            while (rc.spec.replicas != rc.status.readyReplicas && loopNum!=18) {
             loopNum++
             sleep 10
                rc = openshift.selector("rc", "tasks-${dc_version}").object()
            }
            }
            }
        }

      }
    }
    
//     stage('Integration Tests') {
//   steps {
//     echo "Running Integration Tests"
//     script {
//       def status = "000"

//       // Create a new task called "integration_test_1"
//       echo "Creating task"
//       status = sh(returnStdout: true, script: "curl -sw '%{response_code}' -o /dev/null -u 'tasks:redhat1' -H 'Content-Length: 0' -X POST http://tasks.${prefix}-tasks-dev.svc.cluster.local:8080/ws/tasks/integration_test_1").trim()
//       echo "Status: " + status
//       if (status != "201") {
//           error 'Integration Create Test Failed!'
//       }

//       echo "Retrieving tasks"
//       status = sh(returnStdout: true, script: "curl -sw '%{response_code}' -o /dev/null -u 'tasks:redhat1' -H 'Accept: application/json' -X GET http://tasks.${prefix}-tasks-dev.svc.cluster.local:8080/ws/tasks/1").trim()
//       if (status != "200") {
//           error 'Integration Get Test Failed!'
//       }

//       echo "Deleting tasks"
//       status = sh(returnStdout: true, script: "curl -sw '%{response_code}' -o /dev/null -u 'tasks:redhat1' -X DELETE http://tasks.${prefix}-tasks-dev.svc.cluster.local:8080/ws/tasks/1").trim()
//       if (status != "204") {
//           error 'Integration Create Test Failed!'
//       }
//     }
//   }
// }

    // Copy Image to Nexus Docker Registry
    stage('Copy Image to Nexus Docker Registry') {
      steps {
        echo "Copy image to Nexus Docker Registry"
            echo "Copy image to Nexus Docker Registry"
    script {
// Use this for OpenShift 3
        sh "skopeo copy --src-tls-verify=false --dest-tls-verify=false --src-creds openshift:\$(oc whoami -t) --dest-creds admin:redhat docker://docker-registry.default.svc:5000/${devProject}/tasks:${devTag} docker://nexus-registry-gpte-hw-cicd.apps.na311.openshift.opentlc.com/tasks:${devTag}"

      // Tag the built image with the production tag.
      openshift.withCluster() {
        openshift.withProject("${prodProject}") {
          echo "Image ${devProject}/${prodTag}"
          openshift.tag("${devProject}/tasks:${devTag}", "${devProject}/tasks:${prodTag}")
        }
      }
    }

      }
    }
    
    
//    @Alex K:    very nice.  So you ended up executing something similar to the following ? :

// 1)   oc create secret docker-registry nexus-registry-secret \
//    --docker-server=docker://nexus-registry.xyz-nexus.svc.cluster.local \
//    --docker-username=admin \
//    --docker-password=admin123 \
//   -n xyz-tasks-prod

// 2)  oc secrets link default nexus-registry-secret --for=pull  -n xyz-tasks-prod


// oc create secret docker-registry nexus-registry-secret --docker-server=nexus-registry-gpte-hw-cicd.apps.na311.openshift.opentlc.com --docker-username=admin --docker-password=redhat -n 9187-tasks-prod
// oc secrets add serviceaccount/default secrets/<pull_secret_name> --for=pull
    
    stage('Blue/Green Production Deployment') {
  steps {
    echo "Blue/Green Deployment"
    script {
      openshift.withCluster() {
        openshift.withProject("${prodProject}") {
          activeApp = openshift.selector("route", "tasks").object().spec.to.name
          if (activeApp == "tasks-green") {
            destApp = "tasks-blue"
          }
          echo "Active Application:      " + activeApp
          echo "Destination Application: " + destApp

          // Update the Image on the Production Deployment Config
          def dc = openshift.selector("dc/${destApp}").object()
          //dc.spec.template.spec.containers[0].image="image-registry.openshift-image-registry.svc:5000/${devProject}/tasks:${prodTag}"
          // Use this for OpenShift 3
          dc.spec.template.spec.containers[0].image="docker-registry.default.svc:5000/${devProject}/tasks:${prodTag}"
          openshift.apply(dc)

          // Update Config Map in change config files changed in the source
          openshift.selector("configmap", "${destApp}-config").delete()
          def configmap = openshift.create("configmap", "${destApp}-config", "--from-file=./openshift-tasks/configuration/application-users.properties", "--from-file=./openshift-tasks/configuration/application-roles.properties" )

          // Deploy the inactive application.
          openshift.selector("dc", "${destApp}").rollout().latest();

          // Wait for application to be deployed
          def dc_prod = openshift.selector("dc", "${destApp}").object()
          def dc_version = dc_prod.status.latestVersion
          def rc_prod = openshift.selector("rc", "${destApp}-${dc_version}").object()
          echo "Waiting for ${destApp} to be ready"
          def loopNum = 0
          while (rc_prod.spec.replicas != rc_prod.status.readyReplicas && loopNum != 18) {
            loopNum++
            sleep 10
            rc_prod = openshift.selector("rc", "${destApp}-${dc_version}").object()
          }
        }
      }
    }
  }
}

    //113f779880068878a50de4b7df36808c35
    stage('Switch over to new Version') {
      steps {
    echo "Switching Production application to ${destApp}."
    script {
      openshift.withCluster() {
        openshift.withProject("${prodProject}") {
          def route = openshift.selector("route/tasks").object()
          route.spec.to.name="${destApp}"
          openshift.apply(route)
        }
      }
    }
      }
    }
    
  }
  
  
}
}
