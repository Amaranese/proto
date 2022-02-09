pipeline {
    agent any

    triggers {
        cron('15 13 * * *')
    }
    options {
        skipDefaultCheckout(true)
        // Keep the 10 most recent builds
        buildDiscarder(logRotator(numToKeepStr: '10'))
        timestamps()
    }
    environment {
      PATH="/users/shared/jenkins/miniconda3/bin:$PATH"
    }

    stages {

        stage ("Git Checkout"){
            
            steps{
                echo "Pulls code down from repo to local Jenkins work directory"
                checkout scm
            }
        }
        stage('Build environment') {
            
            steps {
                echo "BUILD ENVIRONMENT"
                echo "CREATE, INSTALL CONDA THEN PIP DEPENDENCIES"
                sh '''conda create --yes -n ${BUILD_TAG} python
                      source activate ${BUILD_TAG} 

                      echo '*****CONDA INSTALL*****'
                      conda install --name ${BUILD_TAG}  --file requirements.txt -y

                      echo '*****PIP INSTALL*****'
                      pip install -r piprequirements.txt
                    '''
            }
        }
        stage('Test environment') {
            when {
                    expression { return readFile('controlme.txt').contains('ENVTEST:Y') }
                  }
            steps {
                sh '''source activate ${BUILD_TAG} 
                      pip freeze
                      pip list
                      which pip
                      which python

                      echo 'LISTING ALL INSTALLED PACKAGES : '
                      conda list
                    '''
            }
        }
        stage('Static code metrics') {
            when {
                    expression { return readFile('controlme.txt').contains('STATICTEST:Y') }
                  }
            steps {
                echo "Raw metrics"
                sh  ''' source activate ${BUILD_TAG}
                        radon raw --json testTargets/ > /Users/adammcmurchie/projects/Jenkins-Stuff/proto/testresults/raw_report.json
                        radon cc --json testTargets/ > /Users/adammcmurchie/projects/Jenkins-Stuff/proto/testresults/cc_report.json
                        radon mi --json testTargets/ > /Users/adammcmurchie/projects/Jenkins-Stuff/proto/testresults/mi_report.json
                    '''
            }
        }
        stage('Static code Coverage') {
            when {
                    expression { return readFile('controlme.txt').contains('COVERAGE:Y') }
                  }
            steps {
                echo "Code Coverage"
                sh  ''' source activate ${BUILD_TAG}
                        coverage run testTargets/main.py || true
                        python -m coverage xml -o ./reports/coverage.xml

                    '''
            }
            post{
                always{
                    step([$class: 'CoberturaPublisher',
                                   autoUpdateHealth: false,
                                   autoUpdateStability: false,
                                   coberturaReportFile: 'reports/coverage.xml',
                                   failNoReports: false,
                                   failUnhealthy: false,
                                   failUnstable: false,
                                   maxNumberOfBuilds: 10,
                                   onlyStable: false,
                                   sourceEncoding: 'ASCII',
                                   zoomCoverageChart: false])
                }
            }
        }
        stage('Unit tests') {
            when {
                    expression { return readFile('controlme.txt').contains('UNITTEST:Y') }
                  }
            steps {
                sh  ''' source activate ${BUILD_TAG}
                        python -m pytest --verbose --junit-xml 'results/results.xml' || true

                        echo 'Checking directory for reporty'
                        ls -alh
                    '''
            }
            post {
                always {
                    junit 'results/results.xml'
                }
            }
        }
        stage('Pylint Tests') {
            when {
                    expression { return readFile('controlme.txt').contains('PYLINT:Y') }
                  }
            steps {
                echo "PEP8 style check"
                sh  ''' source activate ${BUILD_TAG}
                        pylint --disable=C testTargets || true
                    '''
            }
        }
    }
    post {
        always {
            sh 'conda remove --yes -n ${BUILD_TAG} --all'
        }
        success {  
             echo "build ${BUILD_TAG} succesfull" 
         }  
         failure {  
             mail bcc: '', body: "<b>Example</b><br>Project: ${env.JOB_NAME} <br>Build Number: ${env.BUILD_NUMBER} <br> URL de build: ${env.BUILD_URL}", cc: '', charset: 'UTF-8', from: '', mimeType: 'text/html', replyTo: '', subject: "ERROR CI: Project name -> ${env.JOB_NAME}", to: "murchie@gmail.com";  
         }


    }
}