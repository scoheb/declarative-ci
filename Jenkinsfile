env.ghprbGhRepository = env.ghprbGhRepository ?: 'CentOS-PaaS-SIG/ci-pipeline'
env.ghprbActualCommit = env.ghprbActualCommit ?: 'master'

def rpmbuildLabel = "stable"
def ostreeLabel   = "stable"

def openshiftProject = "continuous-infra-devel"

def CANNED_CI_MESSAGE = '{"commit":{"username":"zdohnal","stats":{"files":{"README.patches":{"deletions":0,"additions":30,"lines":30},"sources":{"deletions":1,"additions":1,"lines":2},"vim.spec":{"deletions":7,"additions":19,"lines":26},".gitignore":{"deletions":0,"additions":1,"lines":1},"vim-8.0-rhbz1365258.patch":{"deletions":0,"additions":12,"lines":12}},"total":{"deletions":8,"files":5,"additions":63,"lines":71}},"name":"Zdenek Dohnal","rev":"3ff427e02625f810a2cedb754342be44d6161b39","namespace":"rpms","agent":"zdohnal","summary":"Merge branch \'f25\' into f26","repo":"vim","branch":"f26","seen":false,"path":"/srv/git/repositories/rpms/vim.git","message":"Merge branch \'f25\' into f26\\n","email":"zdohnal@redhat.com"},"topic":"org.fedoraproject.prod.git.receive"}'

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
                stage("rpmbuild image build") {
                    when {
                        changeset "config/Dockerfiles/rpmbuild/**"
                    }
                    steps {
                        script {
                            openshift.withCluster() {
                                openshift.withProject(openshiftProject) {
                                    def result = openshift.startBuild("rpmbuild",
                                            //"--commit",
                                            //env.ghprbActualCommit,
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




