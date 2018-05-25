#!groovy

@Library('Reform')
import uk.gov.hmcts.Ansible
import uk.gov.hmcts.Artifactory
import uk.gov.hmcts.Packager
import uk.gov.hmcts.RPMTagger

def artifactory = new Artifactory(this)
def packager = new Packager(this, 'probate')
def ansible = new Ansible(this, 'probate')
def artifactorySourceRepo = "probate-local"
def rpmTagger

properties([
        [$class: 'GithubProjectProperty', displayName: 'Probate Deployment Pipeline', projectUrlStr: 'https://github.com/hmcts/probate-deploy-rpm.git'],
        parameters([
                string(description: 'environment to  deploy(dev,test or demo)', defaultValue: 'dev', name: 'deploy_enviroinment')
        ])
])



node {
    try {
        stage('Checkout') {
            deleteDir()
            checkout scm
            dir('ansible-management') {
                git url: "https://github.com/hmcts/ansible-management", branch: "master", credentialsId: "jenkins-public-github-api-token"
            }
        }

        deploy_enviroinment =  (deploy_enviroinment in ['dev', 'test', 'demo']) 
                                ? deploy_enviroinment : error ("$deploy_enviroinment is not one of reforms deployment environments")
        stage("Install") {
            ansible.runInstallPlaybook('{}', deploy_enviroinment)
        }

        stage("Deploy") {

            ansible.runDeployPlaybook('{}', deploy_enviroinment)
        }

        stage('show Installed RPM') {
           ansible.run("rpm -qa probate*", deploy_enviroinment, 'runShellcommand.yml', 'master', '', 'all', false)        
        }
            
        stage('smoke test') {
            ws('workspace/probate-frontend/build') {
                env.PROBATE_FRONTEND_URL = ""
                if(deploy_enviroinment == "demo") {
                    env.PROBATE_FRONTEND_URL = "https://www." + deploy_enviroinment + ".probate.reform.hmcts.net/"
                        // don't run smoke test bacuse of AAA filtrs
                }
                else {
                    env.PROBATE_FRONTEND_URL = "https://www-" + deploy_enviroinment + ".probate.reform.hmcts.net/"
                    git url: 'git@git.reform.hmcts.net:probate/smoke-tests.git'
                    sh '''
                       npm install
                       npm test
                    '''
                }
                deleteDir()
            }
        }
    } catch (Throwable err) {
        slackSend(
                channel: '#probate-jenkins',
                color: 'danger',
                message: "Deployment To " + deploy_enviroinment +" FAILED")
        throw err
    }
}
