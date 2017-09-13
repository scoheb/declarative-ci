def rpmbuildLabel = "stable"
def ostreeLabel = "stable"

pipeline {
    agent any
    stages {
        stage("Checkout") {
            steps {
                checkout scm
            }
        }
        stage("Container Builds") {
            parallel {
                stage("rpmbuild container") {
                    when {
                        changeset "config/Dockerfiles/rpmbuild/**"
                    }
                    steps {
                        script {
                            openshift.withCluster() {
                                openshift.withProject("continuous-infra") {
                                    def result = openshift.startBuild("rpmbuild",
                                            "--commit",
                                            "refs/pull/281/head",
                                            "--follow",
                                            "--wait")
                                    def out = result.out.trim()
                                    echo "Operation output: " + out

                                    def describeStr = openshift.selector(out).describe()
                                    out = describeStr.out.trim()
                                    echo "Operation output: " + out

                                    def imageHash = sh(
                                            script: "echo \"${out}\" | grep 'Image Digest:' | cut -f2- -d:",
                                            returnStdout: true
                                    ).trim()
                                    echo "imageHash: " + imageHash
                                    def commitID = "123456"

                                    openshift.tag("continuous-infra/rpmbuild@" + imageHash,
                                            "continuous-infra/rpmbuild:" + commitID)

                                    rpmbuildLabel = commitID
                                }
                            }
                        }
                    }
                }
                stage("ostree") {
                    when {
                        changeset "config/Dockerfiles/ostree/**"
                    }
                    steps {
                        script {
                            echo "ostree!"
                            ostreeLabel = "ostree-latest"
                        }
                    }
                }
            }
        }
        stage("Run Stage Job") {
            steps {
                script {
                    echo rpmbuildLabel
                    echo ostreeLabel
                    def build = build job: 'ci-stage-pipeline-f26',
                            parameters: [
                                    string(name: 'CI_MESSAGE', value: CANNED_CI_MESSAGE),
                                    string(name: 'ghprbActualCommit', value: "${params.ghprbActualCommit}"),
                                    string(name: 'ghprbGhRepository', value: "${params.ghprbGhRepository}"),
                                    string(name: 'sha1', value: "${params.sha1}"),
                                    string(name: 'ghprbPullId', value: "${params.ghprbPullId}"),
                                    string(name: 'RPMBUILD_TAG', value: rpmbuildLabel)]
                    wait: true

                    if (build.result == 'SUCCESS') {
                        echo "yay!"
                    } else {
                        error "build failed!"
                    }
                }
            }
        }
    }
}




