/**
 * CI Stage Pipeline Trigger
 *
 * This is a declarative pipeline for the CI stage pipeline
 * that includes the building of images based on PRs
 *
 */

// Defaults for SCM operations
env.ghprbGhRepository = env.ghprbGhRepository ?: 'CentOS-PaaS-SIG/ci-pipeline'
env.ghprbActualCommit = env.ghprbActualCommit ?: 'master'
env.sha1 = env.sha1 ?: 'master'

// Defaults for tagging of images
def rpmbuildLabel = "stable"
def ostreeLabel   = "stable"

// Openshift project
def openshiftProject = "continuous-infra-devel"

def CANNED_CI_MESSAGE = ""

pipeline {
    agent {
      kubernetes {
        cloud 'openshift'
        label 'mypod'
        containerTemplate {
          name 'maven'
          image 'maven:3.3.9-jdk-8-alpine'
          ttyEnabled true
          command 'cat'
        }
      }
    }
    stages {
        stage("Re-Checkout") {
            checkout([
                    $class: 'GitSCM',
                    branches: scm.branches,
                    doGenerateSubmoduleConfigurations: scm.doGenerateSubmoduleConfigurations,
                    extensions: scm.extensions,
                    userRemoteConfigs: scm.userRemoteConfigs
            ])
        }
        stage("Get Changelog") {
            steps {
                node('master') {
                    echo "PR number is: ${env.CHANGE_ID}"
                    echo getChangeString()
                }
                sh 'cat config/Dockerfiles/rpmbuild/Dockerfile'
                sh 'cat config/Dockerfiles/ostree/Dockerfile'
            }
        }
        stage("rpmbuild image build") {
            when {
                // Only build if we have related files in changeset
                changeset "config/Dockerfiles/rpmbuild/**"
            }
            steps {
                script {
                    echo "rpmbuild will build!"
                    rpmbuildLabel = "rpmbuild-latest"
                }
//                        script {
//                            // - build in Openshift
//                            // - startBuild with a commit
//                            // - Get result Build and get imagestream manifest
//                            // - Use that to create a unique tag
//                            // - This tag will then be passed as an image input
//                            //   to the podTemplate/containerTemplate to create
//                            //   our slave pod.
//                            openshift.withCluster() {
//                                openshift.withProject(openshiftProject) {
//                                    def result = openshift.startBuild("rpmbuild",
//                                            // wait until we upgrade to 3.6
//                                            // for next params:
//                                            //"--commit",
//                                            //env.ghprbActualCommit,
//                                            "--wait")
//                                    def out = result.out.trim()
//                                    echo "Resulting Build: " + out
//
//                                    def describeStr = openshift.selector(out).describe()
//                                    out = describeStr.out.trim()
//
//                                    def imageHash = sh(
//                                            script: "echo \"${out}\" | grep 'Image Digest:' | cut -f2- -d:",
//                                            returnStdout: true
//                                    ).trim()
//                                    echo "imageHash: " + imageHash
//
//                                    echo "Creating CI tag for " + openshiftProject +"/rpmbuild: rpmbuild:" + env.ghprbActualCommit
//
//                                    openshift.tag(openshiftProject + "/rpmbuild@" + imageHash,
//                                            openshiftProject + "/rpmbuild:" + env.ghprbActualCommit)
//
//                                    rpmbuildLabel = env.ghprbActualCommit
//                                }
//                            }
//                        }
            }
        }
        stage("ostree image build") {
            when {
                changeset "config/Dockerfiles/ostree/**"
            }
            steps {
                script {
                    echo "ostree will build!"
                    ostreeLabel = "ostree-latest"
                }
            }
        }
        stage("Run Stage Job") {
            steps {
                script {
                    // Use tags derived from above image builds
                    //
                    echo "rpmbuild tag: " + rpmbuildLabel
                    echo "ostree   tag: " + ostreeLabel
                    def build = build job: 'ci-stage-pipeline-f26',
                            parameters: [
                                    string(name: 'CI_MESSAGE', value: CANNED_CI_MESSAGE),
                                    string(name: 'ghprbActualCommit', value: "${env.ghprbActualCommit}"),
                                    string(name: 'ghprbGhRepository', value: "${env.ghprbGhRepository}"),
                                    string(name: 'sha1', value: "${env.sha1}"),
                                    string(name: 'ghprbPullId', value: "${env.ghprbPullId}"),
                                    string(name: 'RPMBUILD_TAG', value: rpmbuildLabel)]
                    wait: true
                }
            }
        }
    }
    post {
        success {
            echo "yay!"
        }
        failure {
            error "build failed!"
        }
    }
}

@NonCPS
def getChangeString() {
    MAX_MSG_LEN = 100
    def changeString = ""

    echo "Gathering SCM changes"
    def changeLogSets = currentBuild.changeSets
    for (int i = 0; i < changeLogSets.size(); i++) {
        def entries = changeLogSets[i].items
        for (int j = 0; j < entries.length; j++) {
            def entry = entries[j]
            truncated_msg = entry.msg.take(MAX_MSG_LEN)
            changeString += " - ${truncated_msg} [${entry.author}]\n"
	    def files = new ArrayList(entry.affectedFiles)
            for (int k = 0; k < files.size(); k++) {
              def file = files[k]
              changeString += " x ${file.path}\n"
            }
        }
    }

    if (!changeString) {
        changeString = " - No new changes\n"
    }
    return changeString
}


