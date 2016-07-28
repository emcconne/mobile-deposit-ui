#!groovy
import groovy.json.JsonSlurper

/**
* The following parameters are used in this pipeline (thus available as groovy variables via Jenkins job parameters):
*/


stage 'build'
    node{
        checkout scm
        tool name: 'ant', type: 'hudson.tasks.Ant$AntInstallation'
        sh 'mvn -DskipTests clean package'
        stash name: 'source', excludes: 'target/'
        archive includes: 'target/*.war'
    }
        
// stage 'test[unit&quality]'
//     parallel \
//     'unit-test': {
//         node {
//             unstash 'source'
//             sh 'mvn -Dmaven.test.failure.ignore=true test'
//             step([$class: 'JUnitResultArchiver', testResults: '**/target/surefire-reports/TEST-*.xml'])
//             if(currentBuild.result == 'UNSTABLE'){
//                 error "Unit test failures"
//             }
//         }
//     }, 
//     'quality-test': {
//         node {
//             unstash 'source'
//             sh 'mvn sonar:sonar'
//         } 
//     }

stage name:'deploy[development]', concurrency:1
    node{
        unstash 'source'
        def octool = tool name: 'oc', type: 'com.cloudbees.plugins.openshift.OpenShiftClient'
        sh "oc -version"
        wrap([$class: 'OpenShiftBuildWrapper', url: 'https://10.2.2.2:8443' , credentialsId: 'da933727-7cad-438f-9fe8-2878a291e83f', insecure: true]) {
            oc('project mobile-development -q')

            def bc = oc('get bc -o json')
            if(!bc.items) {
            	//TODO still a branch problem here
                oc("new-app --name=mobile-deposit-ui --code='.' --image-stream=jboss-webserver30-tomcat8-openshift")
                wait('app=mobile-deposit-ui', 7, 'MINUTES')
                oc('expose service mobile-deposit-ui')
            } else {
                oc("start-build mobile-deposit-ui --from-dir=. --$OS_BUILD_LOG")
            }
        }
    }
    checkpoint 'deploy[development]-complete'

// stage name:'deploy[test]', concurrency:1
//     mail to: 'apemberton@cloudbees.com', subject: "Deploy mobile-deposit-ui version #${env.BUILD_NUMBER} to test?",
//         body: "Deploy mobile-deposit-ui#${env.BUILD_NUMBER} to test and start functional tests? Approve or Reject on ${env.BUILD_URL}."
//     input "Deploy mobile-deposit-ui#${env.BUILD_NUMBER} to test?"
    
//     node{
//         wrap([$class: 'OpenShiftBuildWrapper', url: OS_URL, credentialsId: OS_CREDS_TEST, insecure: true]) {
//             def project = oc('project mobile-development -q')
//             def is = oc('get is -o json')
//             def image = is?.items[0].metadata.name
            
//             oc("tag $image:latest $image:test")

//             project = oc('project mobile-test -q')
//             def dc = oc('get dc -o json')
//             if(!dc.items){
//                 oc("new-app mobile-development/$image:test")
//                 wait('app=mobile-deposit-ui', 7, 'MINUTES')
//                 oc('expose service mobile-deposit-ui')
//             }
            
//             sleep time: 120, unit: 'SECONDS' //give JBoss another minute to start; probably better ways to validate
//         }
//     }
    
// stage 'test[functional]'
//     node {
//         unstash 'source'
//         withCredentials([[$class: 'UsernamePasswordMultiBinding', credentialsId: SAUCE_CREDS, 
//             usernameVariable: 'SAUCE_USER_NAME', passwordVariable: 'SAUCE_API_KEY']]) {
//             sh "mvn verify" //TODO pass URL to test route
//         }
//         step([$class: 'JUnitResultArchiver', testResults: '**/target/failsafe-reports/TEST-*.xml', testDataPublishers: [[$class: 'SauceOnDemandReportPublisher', jobVisibility: 'public']]])
        
//     }
//     checkpoint 'test[functional]-complete'
    
// stage name:'deploy[production]', concurrency:1
//     mail to: 'apemberton@cloudbees.com', subject: "Deploy mobile-deposit-ui version #${env.BUILD_NUMBER} to production?",
//         body: "Deploy mobile-deposit-ui#${env.BUILD_NUMBER} to production? Approve or Reject on ${env.BUILD_URL}."
//     input "Deploy mobile-deposit-ui#${env.BUILD_NUMBER} to production?"
    
//     node{
//         wrap([$class: 'OpenShiftBuildWrapper', url: OS_URL, credentialsId: OS_CREDS_PROD, insecure: true]) {
//             def project = oc('project mobile-development -q')
//             def is = oc('get is -o json')
//             def image = is?.items[0]?.metadata.name
            
//             oc("tag $image:test $image:production")

//             project = oc('project mobile-production -q')
//             def dc = oc('get dc -o json')
//             if(!dc.items){
//                 oc("new-app mobile-development/$image:production")
//                 wait('app=mobile-deposit-ui', 7, 'MINUTES')
//                 oc('expose service mobile-deposit-ui')
//             }
//         }
//     }
/**
* Execute OpenShift v3 'oc' CLI commands, sending output to Jenkins log console and returned 
* to the user as either JSON or a raw string.
* 
* @see: https://docs.openshift.com/enterprise/3.0/cli_reference/index.html 
*/

def oc(cmd){
    def output
    sh "set -o pipefail"
    sh "oc $cmd 2>&1 | tee output.jenkins"
    output = readFile 'output.jenkins'
    if(output.startsWith('{')){
        output = new JsonSlurper().parseText(output)
    }
    sh "rm output.jenkins"
    return output
}

/**
* Check for a pod matching $selector to be in 'Running' status until the given timeout. 
*/
def wait(selector, time, unit){
    timeout(time: time, unit: unit){
        waitUntil{
            sleep 5L //poll only every 5 seconds
            def pod = oc("get pods --selector='$selector' -o json")
            return pod.items[0]?.status?.phase == 'Running'
        }
    }
}
