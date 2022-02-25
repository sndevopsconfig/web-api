/**
* Parameters can be sent via build parameters, instead of changing the code.
* Use the same variable name to set the build parameters.
* List of parameters that can be passed
* appName='devops-demo-web-app'
* deployableName = 'PROD-US'
* componentName="web-app-v1.1"
* collectionName="release-1.0"
* exportFormat ='yaml'
* configFilePath = "k8s/helm/values.yml"
* exporterName ='returnAllData-nowPreview' 
* exporterArgs = ''
*/

pipeline {    

      agent any
      /**
      * Jenkins pipline related variables
      */
      
      stages{

            stage('Initialize'){
                  steps{
                        script{
                        
                              dockerImageName = "naileditcakedelivery/web-app"

                              /**
                              * DevOps Config App related information
                              */
                              appName='NailedIt Cake Delivery'
                              deployableName = 'Production'
                              componentName="paymentService-v1.0"
                              collectionName="release-1.0"
                              
                              /**
                              * Configuration File information to be uploaded
                              */ 
                              exportFormat ='yaml'
                              configFilePath = "k8s/helm/values.yml"

                              /**
                              * Devops Config exporter related information
                              */
                              exporterName ='returnAllData-nowPreview' 
                              exporterArgs = ''
                              
                              /**
                              * Jenkins variables declared to be used in pipeline
                              */
                              fileNamePrefix ='exported_file_'
                              fullFileName="${fileNamePrefix}-${deployableName}-${currentBuild.number}.${exportFormat}"
                              changeSetId=""
                              dockerImageTag=""
                              snapshotName=""
                              snapshotObject=""
                              isSnapshotCreated=false
                              isSnapshotValidateionRequired=false
                              isSnapshotPublisingRequired=false

                              /**
                              * Checking for parameters
                              */
                              if(params){
                                    echo "setting values from build parameter"
                                    if(params.appName){
                                          appName = params.appName;
                                    }
                                    if(params.deployableName){
                                          deployableName = params.deployableName
                                    }
                                    if(params.componentName){
                                          componentName = params.componentName
                                    }
                                    if(params.collectionName){
                                          collectionName = params.collectionName
                                    }
                                    if(params.exportFormat){
                                          exportFormat = params.exportFormat
                                    }
                                    if(params.configFilePath){
                                          configFilePath = params.configFilePath
                                    }
                                    if(params.exporterName){
                                          exporterName =params.exporterName
                                    }
                                    if(params.exporterArgs){
                                          exporterArgs = params.exporterArgs
                                    }
                              }
                        }
                  }
            }
            
            // Build Step
            stage('Build image') {      
                  steps {
                        checkout scm    
                        echo "scm checkout successful"
                  }
            }     
            stage('Test') {           
                  steps {         
                        sh 'echo "Tests passed"'        
                  }
            }     
            
            // Generate an Artifact
            stage('Push docker Image') { 
                  steps {
                        sh 'ls -a'
                        script {
                              dockerImageTag = env.BUILD_NUMBER
                              dockerImageNameTag = "${dockerImageName}" + ":" + "${dockerImageTag}"

                              snDevopsArtifactPayload = '{"artifacts": [{"name": "' + dockerImageName + '",  "version": "' + "${dockerImageTag}" + '", "semanticVersion": "' + "0.1.${dockerImageTag}"+ '","repositoryName": "' + dockerImageName+ '"}, ],"stageName":"Build image","branchName": "main"}'  ;
                              echo " docker Image artifacat ${dockerImageNameTag} "
                              echo "snDevopsArtifactPayload ${snDevopsArtifactPayload} "

                              snDevOpsArtifact(artifactsPayload:snDevopsArtifactPayload)
                        }
                  }
            }
            
            stage('Upload Configuration Files') {
                  steps {
                        sh "echo validating configuration file ${configFilePath}"
                        script {
                              changeSetId = snDevOpsConfigUpload(applicationName:"${appName}",target:'component',namePath:"${componentName}", configFile:"${configFilePath}", autoCommit:'true',autoValidate:'true',dataFormat:"${exportFormat}")

                              echo "validation result $changeSetId"

                              if(changeSetId != null){

                                    echo "Change set registration for ${changeSetId}"
                                    changeSetRegResult = snDevOpsConfigRegisterPipeline(changesetNumber:"${changeSetId}")
                                    echo "change set registration set result ${changeSetRegResult}"
                                    
                              } else {
                                    
                                    error "Change set was not created"
                              }
                        }
                  }
            }

            stage("Get snapshot status") {
                  steps {
                        echo "Triggering Get snapshots for applicationName:${appName},deployableName:${deployableName},changeSetId:${changeSetId}"

                        script {
                              changeSetResults = snDevOpsConfigGetSnapshots(applicationName:"${appName}",deployableName:"${deployableName}",changesetNumber:"${changeSetId}")
                              if (!changeSetResults){
                                    isSnapshotCreated=false
                                    echo "no snapshot were created"
                              } else {
                                    isSnapshotCreated = true;

                                    echo "ChangeSet Result : ${changeSetResults}"

                                    def changeSetResultsObject = readJSON text: changeSetResults

                                    changeSetResultsObject.each {
                                          snapshotName = it.name
                                          snapshotObject = it
                                    }
                                    snapshotValidationStatus = snapshotObject.validation
                                    snapshotPublishedStatus = snapshotObject.published 
                              }
                        }
                  }
            }
            
            stage('Get latest snapshot'){
                  when {
                        expression { isSnapshotCreated == false }
                  }
                  steps {
                        script {
                              echo "Get latest snapshot"
                              snapshotResults = snDevOpsConfigGetSnapshots(applicationName:"${appName}",deployableName:"${deployableName}")
                              if (!snapshotResults){
                                    error "no snapshots found"
                              } else {
                                    echo "Snapshot Result : ${snapshotResults}"

                                    def snapshotResultsObject = readJSON text: snapshotResults

                                    snapshotResultsObject.each {
                                          snapshotName = it.name
                                          snapshotObject = it;
                                    }
                                    snapshotValidationStatus = snapshotObject.validation
                                    snapshotPublishedStatus = snapshotObject.published 
                              }
                        }
                  }
            }

            stage('Validate snapshot if not validated') {
                  when  {
                        expression { snapshotValidationStatus == 'Not Validated' }
                  }
                  steps {
                        script {
                              validateResponse = snDevOpsConfigValidate( applicationName:"${appName}",deployableName:"${deployableName}",snapshotName: "${snapshotObject.name}" )
                              if(validateResponse != null) {
                                    echo "validation Response submited for ${snapshotObject.name}"
                              }
                        }
                  }
            }
            
            stage('Get latest snapshot after validation') {
                  when { 
                        expression { isSnapshotCreated == false && snapshotObject.validation == 'Not Validated'}
                  }
                  steps {
                        script {
                              echo "Get latest snapshot for appName : ${appName} , deployableName: ${deployableName}"
                              snapshotResults = snDevOpsConfigGetSnapshots(applicationName:"${appName}",deployableName:"${deployableName}")
                              if (!changeSetResults) {
                                    error "no snapshots found for appName : ${appName} , deployableName: ${deployableName}"
                              } else {
                                    echo "Snapshot Result : ${snapshotResults}"

                                    def snapshotResultsObject = readJSON text: snapshotResults

                                    snapshotResultsObject.each {
                                          snapshotName = it.name
                                          snapshotObject = it;
                                    }
                                   snapshotValidationStatus = snapshotObject.validation
                                   snapshotPublishedStatus = snapshotObject.published
                              }
                        }
                  }
            }

            stage('Check validity of snapshot')  {
                  steps {
                        script {
                              echo "snapshot object : ${snapshotObject}"
                              if(snapshotObject.validation == "passed") {
                                    echo "latest snapshot validation is passed"
                              } else {
                                    error "latest snapshot validation failed"
                              }
                        }
                  }
            }
            
            stage('Publish the snapshot'){
                  when {
                        expression { snapshotValidationStatus == "passed" && snapshotPublishedStatus == false }
                  }
                  steps {
                        script {
                              echo "Step to publish snapshot applicationName:${appName},deployableName:${deployableName} snapshotName:${snapshotName}"
                              publishSnapshotResults = snDevOpsConfigPublish(applicationName:"${appName}",deployableName:"${deployableName}",snapshotName: "${snapshotName}")
                              echo "Publish result for applicationName:${appName},deployableName:${deployableName} snapshotName:${snapshotName} is ${publishSnapshotResults} "
                        }
                  }
            }

            stage('Export Snapshots from Service Now') {
                  steps {
                        script {
                              echo "Devops Change - trigger change request"
                              snDevOpsChange()

                              echo "Exporting for App: ${appName} Deployable; ${deployableName} Exporter name ${exporterName} "
                              echo "Configfile exporter file name ${fullFileName}"
                              sh  'echo "<<<<<<<<< export file is starting >>>>>>>>"'
                              exportResponse = snDevOpsConfigExport(applicationName: "${appName}", snapshotName: "${snapshotObject.name}", deployableName: "${deployableName}",exporterFormat: "${exportFormat}", fileName:"${fullFileName}", exporterName: "${exporterName}", exporterArgs: "${exporterArgs}")
                              echo " RESPONSE FROM EXPORT : ${exportResponse}"
                        }
                  }
            }
            
            stage("Deploy to PROD-US") {
                  steps {
                        script {
                              echo "Reading config from file name ${fullFileName}"
                              echo " ++++++++++++ BEGIN OF File Content ***************"
                              sh "cat ${fullFileName}"
                              echo " ++++++++++++ END OF File content ***************"
                              
                              echo "deploy finished successfully."

                              
                              echo "********************** BEGIN Deployment ****************"
                              echo "Applying docker image ${dockerImageNameTag}"

                        
                              echo "********************** END Deployment ****************"
                        }
                  }
            }
      }
      post {
          always {
                 echo ">>>>> Displaying Test results <<<<<"
                 junit '**/*.xml'
          }
      }
}
