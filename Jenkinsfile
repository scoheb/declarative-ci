pipeline {
    agent any
    stages {
        stage("One") {
            steps {
                echo "Hello"
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
                }
            }
        }
	}
	}
    }
}




