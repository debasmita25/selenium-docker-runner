pipeline {

    agent any

    parameters {
        choice(name: 'BROWSER', choices: ['chrome','firefox'])
        string(name: 'TEST_SUITE', defaultValue: 'vendor-app.xml')
        string(name: 'THREAD_COUNT', defaultValue: '3')
        choice(name: 'TEST_ENV', choices: ['qa','stage','prod'])
        string(name: 'BROWSER_SCALE', defaultValue: '2')
    }

    environment {
        TEST_IMAGE = "debasmita25/selenium-tests:latest"
    }

    stages {

        stage('Pull Test Image') {
            steps {
                script {
                    if (isUnix()) {
                        sh "docker pull ${TEST_IMAGE}"
                    } else {
                        bat "docker pull %TEST_IMAGE%"
                    }
                }
            }
        }

        stage('Start Grid') {
            steps {
                script {

                    if (isUnix()) {

                        sh """
                        docker compose -f grid.yaml up -d \
                        --scale ${params.BROWSER}=${params.BROWSER_SCALE}
                        """

                    } else {

                        bat """
                        docker compose -f grid.yaml up -d ^
                        --scale %BROWSER%=%BROWSER_SCALE%
                        """

                    }

                }
            }
        }

        stage('Run Tests') {
            steps {
                script {

                    withEnv([
                        "BROWSER=${params.BROWSER}",
                        "THREAD_COUNT=${params.THREAD_COUNT}",
                        "TEST_SUITE=${params.TEST_SUITE}",
                        "TEST_ENV=${params.TEST_ENV}"
                    ]) {

                        if (isUnix()) {

                            sh """
                            docker compose \
                            -f test-suites.yaml \
                            up --abort-on-container-exit
                            """

                        } else {

                            bat """
                            docker compose ^
                            -f test-suites.yaml ^
                            up --abort-on-container-exit
                            """

                        }
                    }

                    // Dynamic TestNG failure detection
                    def suiteName = params.TEST_SUITE.replace('.xml','')
                    def failedFile = "output/${suiteName}/testng-failed.xml"

                    if (fileExists(failedFile)) {
                        error("Failed tests found in ${suiteName}")
                    }

                }
            }
        }

    }
post {
    always {
        script {
            if (isUnix()) {
                sh "docker compose -f grid.yaml down || true"
                sh "docker compose -f test-suites.yaml down || true"
                sh "docker rmi ${TEST_IMAGE} || true"
            } else {
                bat "docker compose -f grid.yaml down || exit 0"
                bat "docker compose -f test-suites.yaml down || exit 0"
                bat "docker rmi %TEST_IMAGE% || exit 0"
            }

            def suiteName = params.TEST_SUITE.replace('.xml', '')
            def reportFile = "output/results"

            publishHTML([
                allowMissing: true,
                alwaysLinkToLastBuild: true,
                keepAll: true,
                reportDir: reportFile,
                reportFiles: 'ExtentReport.html',
                reportName: 'Automation Test Report'
            ])
        }

        archiveArtifacts artifacts: 'output/results/ExtentReport.html',
                         fingerprint: true,
                         followSymlinks: false
    }
}

}