pipeline {
    agent any
    stages {
        stage("One") {
            steps {
                echo "Hello"
		checkout scm
            }
        }
        stage("Two") {
            when {
                changeset "config/Dockerfiles/rpmbuild/**"
            }
            steps {
                script {
                    echo "rpmbuild!"
                }

            }
        }
    }
}



