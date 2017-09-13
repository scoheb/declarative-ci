
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
                    echo "rpmbuild!"
		    rpmbuildLabel = "rpmbuild-latest"
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
		    sleep 60
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




