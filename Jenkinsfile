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
    environment {
        buildArtifactsPath = "build_artifacts/${currentBuild.number}"
        validationResultsPath = ""
    }
    agent any
    /**
    * Jenkins pipline related variables
    */
    stages {
        // Initialize pipeline
        stage('Initialize') {
            steps {
                script {
                    dockerImageName = "kekaichinose/web-app"

                    /**
                    * DevOps Config App related information
                    */
                    appName = 'PaymentDemo'
                    deployableName = 'Production'
                    componentName = "web-api-v1.0"
                    collectionName = "release-1.0"
                    /**
                    * Configuration File information to be uploaded
                    */ 
                    exportFormat = 'yaml'
                    configFilePath = "k8s/helm/values.yml"
                    /**
                    * Devops Config exporter related information
                    */
                    exporterName = 'returnAllData-nowPreview' 
                    exporterArgs = ''
                    /**
                    * Jenkins variables declared to be used in pipeline
                    */
                    exportFileName = "${buildArtifactsPath}/export_file-${appName}-${deployableName}-${currentBuild.number}.${exportFormat}"
                    changeSetId = ""
                    dockerImageTag = ""
                    snapshotName = ""
                    snapshotObject = ""
                    isSnapshotCreated = false
                    isSnapshotValidateionRequired = false
                    isSnapshotPublisingRequired = false
                    
                    buildNumberArfifact = "grefId123"

                    /**
                    * Checking for parameters
                    */
                    if(params) {
                        echo "setting values from build parameter"
                        if(params.appName) {
                                appName = params.appName;
                        }
                        if(params.deployableName) {
                                deployableName = params.deployableName
                        }
                        if(params.componentName) {
                                componentName = params.componentName
                        }
                        if(params.collectionName) {
                                collectionName = params.collectionName
                        }
                        if(params.exportFormat) {
                                exportFormat = params.exportFormat
                        }
                        if(params.configFilePath) {
                                configFilePath = params.configFilePath
                        }
                        if(params.exporterName) {
                                exporterName = params.exporterName
                        }
                        if(params.exporterArgs) {
                                exporterArgs = params.exporterArgs
                        }
                    }
                }
                echo """---- Build Parameters ----
                applicationName: ${appName}
                namePath: ${componentName}
                configFile: ${configFilePath}
                dataFormat: ${exportFormat}
                """
            }
        }
            
        // Build and publish application image
        stage('Build') {      
            steps {
                checkout scm    
                echo "scm checkout successful"
                
                script {
                    dockerImageTag = env.BUILD_NUMBER
                    dockerImageNameTag = "${dockerImageName}" + ":" + "${dockerImageTag}"

                    snDevopsArtifactPayload = '{"artifacts": [{"name": "' + dockerImageName + '",  "version": "' + "${dockerImageTag}" + '", "semanticVersion": "' + "0.1.${dockerImageTag}"+ '","repositoryName": "' + dockerImageName+ '"}, ],"stageName":"Build image","branchName": "main"}'  ;
                    echo "Docker image artifact: ${dockerImageNameTag} "
                    echo "snDevopsArtifactPayload: ${snDevopsArtifactPayload} "

                    snDevOpsArtifact(artifactsPayload:snDevopsArtifactPayload)
                }
            }
        } 
            
        // Validate code and config data
        stage('Validate') {
            parallel {
                // Validate configuration data changes
                stage('Config') {
                    stages('Config Steps') {
                        // Upload configuration data to DevOps Config
                        stage('Upload') {
                            steps {
                                sh "echo updaing configfile with build number"
                                sh "sed -i 's/${buildNumberArfifact}/${BUILD_NUMBER}/g' ${configFilePath}"
                                sh "echo uploading and auto-validating configuration file: ${configFilePath}"
                                script {
                                    changeSetId = snDevOpsConfigUpload(
                                        applicationName: "${appName}",
                                        target: 'component',
                                        namePath: "${componentName}",
                                        configFile: "${configFilePath}",
                                        autoCommit: 'true',
                                        autoValidate: 'true',
                                        dataFormat: "${exportFormat}"
                                    )

                                    echo "Changeset: $changeSetId created"

                                    if(changeSetId != null) {
                                        // // DevOps Change Enable
                                        echo "Register changeset: ${changeSetId} to pipeline"
                                        changeSetRegResult = snDevOpsConfigRegisterPipeline(
                                            applicationName: "${appName}",
                                            changesetNumber: "${changeSetId}"
                                        )
                                        echo "Pipeline registration result: ${changeSetRegResult}"
                                        //
                                    } else {
                                        error "Changeset was not created"
                                    }
                                }
                            }
                        }

                        // Auto-validation was set during upload; get status of snapshot
                        stage('Validate') {
                            steps {
                                echo "Triggering snDevOpsConfigGetSnapshots for applicationName:${appName},deployableName:${deployableName},changeSetId:${changeSetId}"

                                script {
                                    changeSetResults = snDevOpsConfigGetSnapshots(
                                        applicationName:"${appName}",
                                        deployableName:"${deployableName}",
                                        changesetNumber:"${changeSetId}",
                                        showResults: false,
                                        markFailed: false
                                    )
                                    if (!changeSetResults){
                                        isSnapshotCreated=false

                                        echo "No snapshots were created"
                                    } else {
                                        isSnapshotCreated = true;
                                        
                                        echo "Changeset result : ${changeSetResults}"

                                        def changeSetResultsObject = readJSON text: changeSetResults

                                        changeSetResultsObject.each {
                                            snapshotName = it.name
                                            snapshotObject = it
                                        }
                                        snapshotValidationStatus = snapshotObject.validation
                                        snapshotPublishedStatus = snapshotObject.published 
                                    }
                                }

                                script {
                                    echo "Snapshot object : ${snapshotObject}"

                                    validationResultsPath = "${snapshotName}_${currentBuild.projectName}_${currentBuild.number}.xml"
                                    
                                    if(snapshotObject.validation == "passed" || snapshotObject.validation == "passed_with_exception") {
                                        echo "Latest snapshot passed validation"
                                    } else {
                                        error "Latest snapshot failed"
                                    }
                                }
                            }
                        }

                        // Publish snapshot now that it passed validation
                        stage('Publish') {
                            when {
                                expression { (snapshotValidationStatus == "passed" || snapshotValidationStatus == "passed_with_exception") && snapshotPublishedStatus == false }
                            }
                            steps {
                                script {
                                    echo "Step to publish snapshot applicationName:${appName},deployableName:${deployableName} snapshotName:${snapshotName}"
                                    publishSnapshotResults = snDevOpsConfigPublish(applicationName:"${appName}",deployableName:"${deployableName}",snapshotName: "${snapshotName}")
                                    echo "Publish result for applicationName:${appName},deployableName:${deployableName} snapshotName:${snapshotName} is ${publishSnapshotResults} "
                                }
                            }
                        }

                        // Export published snapshot to be used by downstream deployment tools
                        stage('Export') {
                            steps {
                                script {
                                    echo "Exporting config data for App: ${appName}, Deployable: ${deployableName}, Exporter: ${exporterName} "
                                    echo "Export file name ${exportFileName}"
                                    // create build artifacts dir if not created yet
                                    sh "mkdir -p ${buildArtifactsPath}"
                                    
                                    echo "<<<<<<<<< Starting config data export >>>>>>>>"
                                    exportResponse = snDevOpsConfigExport(
                                            applicationName: "${appName}",
                                            snapshotName: "${snapshotObject.name}",
                                            deployableName: "${deployableName}",
                                            exporterFormat: "${exportFormat}",
                                            fileName: "${exportFileName}",
                                            exporterName: "${exporterName}",
                                            exporterArgs: "${exporterArgs}"
                                    )
                                    echo "RESPONSE FROM EXPORT : ${exportResponse}"
                                }
                            }
                        }
                    }
                }

                // Validate application code changes (SIMULATED)
                stage('Code') { 
                    stages {
                        stage('jUnit Test'){ 
                            steps {
                                echo "Running unit tests..."
                            }
                        }
                        
                        stage('SonarQube analysis') {
                            steps {
                                echo "Running code quality analysis..."
                            }
                        }
                    }    
                }
            }
        }

        // Deploy configuration data to UAT environment
        stage('UAT Deployment') {
            steps {
                sleep(time:5,unit:"SECONDS")
            }
        }
        
        // Run functional tests
        stage ('Functional Testing') {
            parallel {      
                stage('Selenium API') { 
                    steps {
                        echo "Selenium API..2..3..4"
                        sleep(time:5,unit:"SECONDS")
                        echo "Selenium API..2..3..4"
                    }
                }
                stage('Selenium UI') {
                    steps {
                        echo "Selenium UI..2..3..4"
                        sleep(time:7,unit:"SECONDS")
                        echo "Selenium API..2..3..4"
                    }
                }
            }
        }
        
        // Submit change management review
        stage('Change Management') {
            steps {
                script {
                    // // DevOps Change Enable
                    echo "DevOps Change - trigger basic change request"
                    /* San Diego
                    snDevOpsChange()
                    */
                    // Tokyo
                    snDevOpsChange(
                            applicationName: "${appName}",
                            snapshotName: "${snapshotName}"
                    )
                    //
                    /*echo "DevOps Change  - trigger advanced change request"
                    snDevOpsChange(changeRequestDetails: """{
                            "setCloseCode": false,
                            "attributes": {
                                "category": "DevOps",
                                "priority": "3",
                                "cmdb_ci": {
                                    "name": "Servers - PaymentDemo - Production"
                                },
                                "business_service": {
                                    "name": "PaymentDemo_Production_1"
                                }
                            }
                    }""")
                    */
                }     
            }
        }

        // Deploy application code and configuration data to production environment
        stage('Deploy to Production') {
                steps {
                    script {
                            echo "Show exported config data from file name ${exportFileName}"
                            echo " ++++++++++++ BEGIN OF File Content ***************"
                            sh "cat ${exportFileName}"
                            echo " ++++++++++++ END OF File content ***************"
                            echo "Exported config data handed off to deployment tool"
                            echo "********************** BEGIN Deployment ****************"
                            echo "Applying docker image ${dockerImageNameTag}"
                            echo "********************** END Deployment ****************"
                    }
                }
        }
    }
    // NOTE: attach policy validation results to run (if the snapshot fails validation)
    post {
        always {
            // create tests dir
            sh "mkdir -p ${buildArtifactsPath}/tests"
            // move policy validation results to build artifacts folder
            sh "mv ${validationResultsPath} ${buildArtifactsPath}/tests/${validationResultsPath}"
            // attach policy validation results
            echo ">>>>> Displaying Test results <<<<<"
            junit testResults: "${buildArtifactsPath}/tests/${validationResultsPath}", skipPublishingChecks: true
        }
    }
}
