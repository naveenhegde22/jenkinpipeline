def getProjectName() {
    return 'JenkinsPipeline'
}

def getJDKVersion() {
    return 'jdk1.8.0_101'
}

def getMavenConfig() {
    return 'maven-config'
}

def getMavenLocation() {
    return 'M2_HOME'
}

def getEnvironment() {
    return  'QA\n' +
            'STG\n' +
            'PRD'
}

def getEmailRecipients() {
    return ''
}

def getReportZipFile() {
    return "Reports_Build_${BUILD_NUMBER}.zip"
}

def getS3ReportPath() {
    return "$projectName"
}

def publishHTMLReports(reportName) {
    // Publish HTML reports (HTML Publisher plugin)
    publishHTML([allowMissing         : true,
                 alwaysLinkToLastBuild: true,
                 keepAll              : true,
                 reportDir            : 'target\\view',
                 reportFiles          : 'index.html',
                 reportName           : reportName])
}

// To be determined dynamically later
def EXECUTOR_AGENT=null

/* Declarative pipeline must be enclosed within a pipeline block */
pipeline {
    // agent section specifies where the entire Pipeline will execute in the Jenkins environment
    agent {
        /**
         * node allows for additional options to be specified
         * you can also specify label '' without the node option
         * if you want to execute the pipeline on any available agent use the option 'agent any'
         */
        node {
            label '' // Execute the Pipeline on an agent available in the Jenkins environment with the provided label
        }
    }

    /**
     * tools to auto-install and put on the PATH
     * some of the supported tools - maven, jdk, gradle
     */
    tools {
        jdk 'jdk1.7'
        maven 'mvn3.5.0'
    }

    /**
     * parameters directive provides a list of parameters which a user should provide when triggering the Pipeline
     * some of the valid parameter types are booleanParam, choice, file, text, password, run, or string
     */
    parameters {
        choice(choices: "$environment", description: '', name: 'ENVIRONMENT')
        string(defaultValue: "$emailRecipients",
                description: 'List of email recipients',
                name: 'EMAIL_RECIPIENTS')
    }

    /**
     * stages contain one or more stage directives
     */
    stages {
        /**
         * the stage directive should contain a steps section, an optional agent section, or other stage-specific directives
         * all of the real work done by a Pipeline will be wrapped in one or more stage directives
         */
        stage('Prepare') {
            steps {
                // GIT submodule recursive checkout
                checkout scm: [
                        $class: 'GitSCM',
                        branches: scm.branches,
                        doGenerateSubmoduleConfigurations: false,
                        extensions: [[$class: 'SubmoduleOption',
                                      disableSubmodules: false,
                                      parentCredentials: false,
                                      recursiveSubmodules: true,
                                      reference: '',
                                      trackingSubmodules: false]],
                        submoduleCfg: [],
                        userRemoteConfigs: scm.userRemoteConfigs
                ]
                // copy managed files to workspace
                script {
                    if(params.USE_INPUT_DUNS) {
                        configFileProvider([configFile(fileId: '609999e4-446c-4705-a024-061ed7ca2a11',
                                targetLocation: 'input/')]) {
                            echo 'Managed file copied to workspace'
                        }
                    }
                }
            }
        }

        stage('User Input') {
            // when directive allows the Pipeline to determine whether the stage should be executed depending on the given condition
            // built-in conditions - branch, expression, allOf, anyOf, not etc.
            when {
                // Execute the stage when the specified Groovy expression evaluates to true
                expression {
                    return params.ENVIRONMENT ==~ /(?i)(STG|PRD)/
                }
            }
            /**
             * steps section defines a series of one or more steps to be executed in a given stage directive
             */
            steps {
                // script step takes a block of Scripted Pipeline and executes that in the Declarative Pipeline
                script {
                    // This step pauses Pipeline execution and allows the user to interact and control the flow of the build.
                    // Only a basic "process" or "abort" option is provided in the stage view
                    input message: '', ok: 'Proceed',
                            parameters: [
                                    string(name: '', description: ''),
                            ]
                }
            }
        }

        stage('Maven version') {
            steps {
                echo "Hello, Maven"
                sh "mvn --version" // Runs a Bourne shell script, typically on a Unix node
            }
        }

        stage('Maven Install') {
            steps {
                script {
                    /**
                     * Override maven in this stage
                     * you can also use 'tools {maven: "$mavenLocation"}'
                     *
                     * globalMavenSettingsConfig: Select a global maven settings element from File Config Provider
                     * jdk: Select a JDK installation
                     * maven: Select a Maven installation
                     */
                    withMaven(globalMavenSettingsConfig: "$mavenConfig", jdk: "$JDKVersion", maven: "$mavenLocation") {
                        /**
                         * To proceed to the next stage even if the current stage failed,
                         * enclose this stage in a try-catch block
                         */
                        try {
                            sh "mvn clean install"
                        } catch (Exception err) {
                            echo 'Maven clean install failed'
                            currentBuild.result = 'FAILURE'
                        } finally {
                            publishHTMLReports('Reports')
                        }
                    }
                }
            }
        }

        stage('Quality Analysis') {
            steps {
                /**
                 * makes use of one single agent, and spins off 2 runs of the steps inside each parallel branch
                 */
                parallel(
                        "Integration Test": {
                            echo 'Run integration tests'
                        },
                        "Sonar Scan": {
                            sh "mvn sonar:sonar"
                        }
                )
            }
        }

        stage('Transfer Script') {
            steps {
                // provide SSH credentials to builds via a ssh-agent in Jenkins
                sshagent(['test-key']) {
                    sh "scp test.sh ${identity}:${path}"
                }
            }
        }

        stage('Execute Script') {
            steps {
                script {
                    // provide SSH credentials to builds via a ssh-agent in Jenkins
                    sshagent(['test-key']) {
                        // AWS credentials binding - each binding will define an environment variable active within the scope of the step
                        withCredentials([[$class: 'AmazonWebServicesCredentialsBinding',
                                          accessKeyVariable: 'AWS_ACCESS_KEY',
                                          credentialsId: 'test',
                                          secretKeyVariable: 'AWS_SECRET_KEY']]) {
                            sh """
                                ssh ${identity} "chmod 755 test.sh && ./test.sh --aws-access-key ${AWS_ACCESS_KEY} " \
                                    "--aws-secret-key ${AWS_SECRET_KEY}"
                            """
                        }
                    }
                }
            }
        }

        stage('Determine Agent') {
            steps {
                script {
                    if(params.ENVIRONMENT == 'PRD') {
                        EXECUTOR_AGENT="prd-jenkins-slave"
                    } else {
                        EXECUTOR_AGENT="devops-jenkins-slave"
                    }
                }
                /**
                 * Saves a set of files for use later in the same build, generally on another node/workspace.
                 * Stashed files are not otherwise available and are generally discarded at the end of the build.
                 * Note that the stash and unstash steps are designed for use with small files.
                 * For large data transfers, use the External Workspace Manager plugin.
                 */
                stash 'workspace'
            }
        }

        stage('Deploy') {
            steps {
                /**
                 * Execute the steps in the dynamically determined node
                 * We don't use 'agent' here as it won't have access to the script context
                 * i.e.
                 * agent {
                 *     label EXECUTOR_AGENT
                 * }
                 * will result in error as EXECUTOR_AGENT will be 'null'
                 */
                node(EXECUTOR_AGENT) {
                    // Restores a set of files previously stashed into the current workspace in a different node
                    unstash('workspace')
                    script {
                        withAWS(profile: 'Deploy-User') {
                            sh "python deploy.py"
                        }
                    }
                }
            }
        }

        stage('Collect Reports') {
            steps {
                echo "Reports directory: ${workspace}/target/view"
                zip dir: "${workspace}/target", zipFile: "$reportZipFile" // Create a zip file of content in the workspace
            }
        }

        stage('Archive reports in S3') {
            steps {
                // withAWS step provides authorization for the nested steps
                withAWS(region: 'us-east-1', profile: '') {
                    // Upload a file/folder from the workspace to an S3 bucket
                    s3Upload(file: "$reportZipFile",
                            bucket: '',
                            path: "$s3ReportPath")
                }
            }
        }
    }

    /**
     * post section defines actions which will be run at the end of the Pipeline run or stage
     * post section condition blocks: always, changed, failure, success, unstable, and aborted
     */
    post {
        // Run regardless of the completion status of the Pipeline run
        always {
            // send email
            // email template to be loaded from managed files
            emailext body: '${SCRIPT,template="managed:EmailTemplate"}',
                    attachLog: true,
                    compressLog: true,
                    attachmentsPattern: "$reportZipFile",
                    mimeType: 'text/html',
                    subject: "Pipeline Build ${BUILD_NUMBER}",
                    to: "${params.EMAIL_RECIPIENTS}"

            // clean up workspace
            deleteDir()
        }
        // Only run the steps if the current Pipeline’s or stage’s run has a "success" status
        success {
            script {
                def payload = """
{
    "@type": "MessageCard",
    "@context": "http://schema.org/extensions",
    "themeColor": "0076D7",
    "summary": "Test Message",
    "sections": [{
        "activityTitle": "Test Message",
        "activitySubtitle": "",
        "activityImage": "https://cwiki.apache.org/confluence/download/attachments/30737784/oozie_47x200.png",
        "facts": [{
            "name": "ENVIRONMENT",
            "value": "${params.ENVIRONMENT}"
        }],
        "markdown": true
    }],
    "potentialAction": [{
        "@type": "OpenUri",
        "name": "View Build",
        "targets": [
            { "os": "default", "uri": "${BUILD_URL}" }
        ]
    }]
}"""
                // publish message to webhook
                httpRequest httpMode: 'POST',
                        acceptType: 'APPLICATION_JSON',
                        contentType: 'APPLICATION_JSON',
                        url: "${webhookUrl}",
                        requestBody: payload
            }
        }
    }

    // configure Pipeline-specific options
    options {
        // keep only last 10 builds
        buildDiscarder(logRotator(numToKeepStr: '10'))
        // timeout job after 60 minutes
        timeout(time: 60,
                unit: 'MINUTES')
    }

}