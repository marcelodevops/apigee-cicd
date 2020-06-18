#!groovy

@Library('slackNotifications-shared-library@master') _

pipeline
{
    agent any
    tools {
        maven 'M3'
        jdk 'JDK'
        nodejs 'NODEJS'
    }

    environment {
        //getting the current stable/deployed revision...this is used in undeloy.sh in case of failure...
        stable_revision = sh(
            script: 'curl -H "Authorization: Basic $base64encoded" "https://api.enterprise.apigee.com/v1/organizations/marcelodevopsgarcia-eval/apis/hr-api/deployments" | jq -r ".environment[0].revision[0].name"', returnStdout: true).trim()
    }

    stages {
        stage('Initial-Checks') {
            /* groovylint-disable-next-line SpaceAfterClosingBrace */
            steps {
                sendNotifications 'STARTED'
                sh 'npm -v'
                sh 'mvn -v'
                echo "$apigeeUsername"
                echo "Stable Revision: ${env.stable_revision}"
        }}
        stage('Policy-Code Analysis') {
            steps {
                sh 'sudo npm install -g apigeelint'
                sh 'apigeelint -s hr-api/apiproxy/ -f codeframe.js'
            }
        }
        stage('Unit-Test-With-Coverage') {
            steps {
                script {
                    /* groovylint-disable-next-line NestedBlockDepth */
                    try {
                        sh 'npm install'
                        sh 'npm test test/unit/*.js'
                        sh 'npm run coverage test/unit/*.js'
                    } catch (e) {
                        throw e
                    } finally {
                        //sh "cd coverage && cp cobertura-coverage.xml $WORKSPACE"
                        //step([$class: 'CoberturaPublisher', coberturaReportFile: 'cobertura-coverage.xml'])
                    }
                }
            }
        }
        /*stage('Promotion') {
            steps {
                timeout(time: 2, unit: 'DAYS') {
                    input 'Do you want to Approve?'
                }
            }
        }*/
        stage('Deploy to Production') {
            steps {
                 //deploy using maven plugin

                 // deploy only proxy and deploy both proxy and config based on edge.js update
                //sh "sh && sh deploy.sh"
                sh "mvn -f hr-api/pom.xml install -Pprod -Dusername=${apigeeUsername} -Dpassword=${apigeePassword} -Dapigee.config.options=update"
            }
        }
        stage('Integration Tests') {
            steps {
                script {
                    try {
                        // using credentials.sh to get the client_id and secret of the app..
                        // thought of using them in cucumber oauth feature
                        // sh "sh && sh credentials.sh"
                        sh "cd $WORKSPACE/test/integration && npm install"
                        sh "cd $WORKSPACE/test/integration && npm test"
                    } catch (e) {
                        //if tests fail, I have used an shell script which has 3 APIs to undeploy, 
                        //delete current revision & deploy previous stable revision
                        sh "sh && sh undeploy.sh"
                        throw e
                    } finally {
                        // generate cucumber reports in both Test Pass/Fail scenario
                        sh "cd $WORKSPACE/test/integration && cp reports.json $WORKSPACE"
                        /* groovylint-disable-next-line SpaceAroundMapEntryColon */
                        cucumber fileIncludePattern: 'reports.json'
                        /* groovylint-disable-next-line SpaceAroundMapEntryColon */
                        build job: 'cucumber-report'
                    }
                }
            }
        }
    }

    post {
        always {
            // cucumberSlackSend channel: 'apigee-cicd', json: '$WORKSPACE/reports.json'
            sendNotifications currentBuild.result
        }
    }
}

/*

using shared library for slack reporting
    the lib groovy script must be placed in a vars folder in SCM
using build job to call a Freestyle project which sends the Cucumber reports to slack
    currently the cucumberSlackSend channel: 'apigee-cicd', json: '$WORKSPACE/reports.json' 
        option doesnt send the reports to Slack
*/
