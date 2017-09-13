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
                stage("rpmbuild") {
                    when {
                        changeset "config/Dockerfiles/rpmbuild/**"
                    }
                    steps {
                        script {
                            openshift.withCluster() {
                                openshift.withProject("continuous-infra") {
                                    def result = openshift.startBuild("rpmbuild", "--commit", "refs/pull/281/head", "--wait")
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

                                    openshift.tag("continuous-infra/rpmbuild@" + imageHash, "continuous-infra/rpmbuild:" + commitID)
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
        stage("Run Stage") {
            steps {
                script {
                    echo rpmbuildLabel
                    echo ostreeLabel
                }
            }
        }
    }
}




