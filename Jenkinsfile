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

// Defaults for tagging of images
def rpmbuildLabel = "stable"
def ostreeLabel   = "stable"

// Openshift project
def openshiftProject = "continuous-infra-devel"

// CI_MESSAGE known to build successfully
def CANNED_CI_MESSAGE = '{"commit":{"username":"zdohnal","stats":{"files":{"README.patches":{"deletions":0,"additions":30,"lines":30},"sources":{"deletions":1,"additions":1,"lines":2},"vim.spec":{"deletions":7,"additions":19,"lines":26},".gitignore":{"deletions":0,"additions":1,"lines":1},"vim-8.0-rhbz1365258.patch":{"deletions":0,"additions":12,"lines":12}},"total":{"deletions":8,"files":5,"additions":63,"lines":71}},"name":"Zdenek Dohnal","rev":"3ff427e02625f810a2cedb754342be44d6161b39","namespace":"rpms","agent":"zdohnal","summary":"Merge branch \'f25\' into f26","repo":"vim","branch":"f26","seen":false,"path":"/srv/git/repositories/rpms/vim.git","message":"Merge branch \'f25\' into f26\\n","email":"zdohnal@redhat.com"},"topic":"org.fedoraproject.prod.git.receive"}'

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
        stage("Checkout") {
            steps {
	        echo "${env.STAGE_NAME}"
                checkout scm
		echo getChangeString()
            }
        }
        stage("Image Builds") {
            // Parallel the execution of building CI images
            parallel {
                stage("rpmbuild image build") {
                    when {
                        // Only build if we have related files in changeset
                        changeset "config/Dockerfiles/rpmbuild/**"
                    }
                    steps {
                        script {
                            // - build in Openshift
                            // - startBuild with a commit
                            // - Get result Build and get imagestream manifest
                            // - Use that to create a unique tag
                            // - This tag will then be passed as an image input
                            //   to the podTemplate/containerTemplate to create
                            //   our slave pod.
                            openshift.withCluster() {
                                openshift.withProject(openshiftProject) {
                                    def result = openshift.startBuild("rpmbuild",
                                            // wait until we upgrade to 3.6
                                            // for next params:
                                            //"--commit",
                                            //env.ghprbActualCommit,
                                            "--wait")
                                    def out = result.out.trim()
                                    echo "Resulting Build: " + out

                                    def describeStr = openshift.selector(out).describe()
                                    out = describeStr.out.trim()

                                    def imageHash = sh(
                                            script: "echo \"${out}\" | grep 'Image Digest:' | cut -f2- -d:",
                                            returnStdout: true
                                    ).trim()
                                    echo "imageHash: " + imageHash

                                    echo "Creating CI tag for " + openshiftProject +"/rpmbuild: rpmbuild:" + env.ghprbActualCommit

                                    openshift.tag(openshiftProject + "/rpmbuild@" + imageHash,
                                            openshiftProject + "/rpmbuild:" + env.ghprbActualCommit)

                                    rpmbuildLabel = env.ghprbActualCommit
                                }
                            }
                        }
                    }
                }
                stage("ostree image build") {
                    when {
                        changeset "config/Dockerfiles/ostree/**"
                    }
                    steps {
                        script {
                            echo "ostree TODO"
                            ostreeLabel = "ostree-latest"
                        }
                    }
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
        }
    }

    if (!changeString) {
        changeString = " - No new changes"
    }
    return changeString
}


