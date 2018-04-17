#!groovy
import groovy.json.JsonSlurperClassic
node {

    def BUILD_NUMBER=env.BUILD_NUMBER
    def RUN_ARTIFACT_DIR="tests/${BUILD_NUMBER}"
    def SFSO_USERNAME
    def UT_RUNID

    stage('Checkout Source') {
        // when running in multi-branch job, one must issue this command
        checkout scm
		sh 'echo $BRANCH_NAME'
    }

	if(env.BRANCH_NAME == "dev") {
		withCredentials([string(credentialsId: 'CONNECTED_APP_CONSUMER_KEY', variable: 'CONNECTED_APP_CONSUMER_KEY'), string(credentialsId: 'HUB_ORG', variable: 'HUB_ORG'), string(credentialsId: 'SFDC_HOST_DH', variable: 'SFDC_HOST'), file(credentialsId: 'JWT_KEY_FILE', variable: 'jwt_key_file')]) {

			stage('Create Scratch Org') {
				rc = sh returnStatus: true, script: "sfdx force:auth:jwt:grant --clientid ${CONNECTED_APP_CONSUMER_KEY} --username ${HUB_ORG} --jwtkeyfile \"${jwt_key_file}\" --setdefaultdevhubusername --instanceurl ${SFDC_HOST}"
				if (rc != 0) {
					error 'hub org authorization failed' 
				}

				// need to pull out assigned username
				rmsg = sh returnStdout: true, script: "sfdx force:org:create --definitionfile config/project-scratch-def.json --json --setdefaultusername"
				def jsonSlurper = new JsonSlurperClassic()
				def robj = jsonSlurper.parseText(rmsg)
				if (robj.status != 0) {
					error "org creation failed: ${robj.message}" 
				}
				SFSO_USERNAME=robj.result.username
				robj = null
			}

			try {
				stage('Push To Test Org') {
					rc = sh returnStatus: true, script: "sfdx force:source:push --targetusername ${SFSO_USERNAME}"
					if (rc != 0) {
						error 'push failed'
					}
					// assign permset
					rc = sh returnStatus: true, script: "sfdx force:user:permset:assign --targetusername ${SFSO_USERNAME} --permsetname DreamHouse"
					if (rc != 0) {
						error 'permset:assign failed'
					}
				}

				stage('Run Apex Test') {
					sh "mkdir -p ${RUN_ARTIFACT_DIR}"
					timeout(time: 120, unit: 'SECONDS') {
						rmsg = sh returnStdout: true, script: "sfdx force:apex:test:run --testlevel RunLocalTests --outputdir ${RUN_ARTIFACT_DIR} --targetusername ${SFSO_USERNAME} --json"
						def jsonSlurper = new JsonSlurperClassic()
						def robj = jsonSlurper.parseText(rmsg)
						if (robj.status != 0) {
							error "apex test run failed: ${robj.message}" 
						}
						UT_RUNID=robj.result.testRunId
						robj = null
					}
				}

				stage('Collect Results') {
					timeout(time: 120, unit: 'SECONDS') {
						rmsg = sh returnStdout: true, script: "sfdx force:apex:test:report --testrunid ${UT_RUNID} --outputdir ${RUN_ARTIFACT_DIR} --json 2>&1"
						def jsonSlurper = new JsonSlurperClassic()
						def robj = jsonSlurper.parseText(rmsg)
						def testsFile = new File("${RUN_ARTIFACT_DIR}/test-result-${UT_RUNID}.json")
						if (testsFile.exists()){
							echo testsFile.text
						}
						if (robj.status == 100) {
							error "the build ${BUILD_NUMBER} has failed due to Apex Tests"
						}
						else if (robj.status != 0) {
							error "test:report failed: ${robj.message}"
						}
					}
				}
			}
			catch (err) {
				echo "${err}"
				currentBuild.result = 'FAILURE'
			}
			finally {
				stage('Delete Test Org') {
					rc = sh returnStatus: true, script: "sfdx force:org:delete --targetusername ${SFSO_USERNAME} --noprompt"
					if (rc != 0) {
						error 'org deletion request failed'
					}
				}
			}
		}
	}
	else if(env.BRANCH_NAME == "staging") {
		withCredentials([string(credentialsId: 'CONNECTED_APP_CONSUMER_KEY_TPO', variable: 'CONNECTED_APP_CONSUMER_KEY'), string(credentialsId: 'TPO_ORG', variable: 'TPO_ORG'), file(credentialsId: 'JWT_KEY_FILE', variable: 'jwt_key_file')]) {

        stage('Authorize Org') {
            rc = sh returnStatus: true, script: "sfdx force:auth:jwt:grant --clientid ${CONNECTED_APP_CONSUMER_KEY} --username ${TPO_ORG} --jwtkeyfile \"${jwt_key_file}\""
            if (rc != 0) {
                error 'org authorization failed' 
            }
        }

	    stage('Convert Source to Metadata API Format') {
			sh "mkdir mdapioutput"
	        rc = sh returnStatus: true, script: "force:source:convert --outputdir mdapioutput/"
	        if (rc != 0) {
	            error 'convert failed'
	        }
	    }

		stage('Deploy To The Org') {
			//run a check-only deployment
	        rc = sh returnStatus: true, script: "sfdx force:mdapi:deploy --deploydir mdapioutput/ --targetusername ${TPO_ORG} --checkonly true --wait 100 --verbose"
	        if (rc != 0) {
	            error 'check-only deployment failed'
	        }
		}

		stage('Assign Permset') {
			rc = sh returnStatus: true, script: "sfdx force:user:permset:assign --targetusername ${SFSO_USERNAME} --permsetname DreamHouse"
	        if (rc != 0) {
	            error 'permset:assign failed'
	        }

			//clean up by deleting the Metadata API folder
			sh "rm -rf mdapioutput"
		}
	}
}