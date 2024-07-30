pipeline {
    agent any

    environment {
        AWS_REGION = 'us-east-1' // specify your region
    }

    parameters {
        string(name: 'CFT_TEMPLATE', defaultValue: 'NA', description: 'Enter CloudFormation Template')
        choice(name: 'ACTION', choices: ['create', 'update', 'delete'], description: 'Select Action')
    }

    stages {
        
        stage('Validate CloudFormation Template') {
            steps {
                script {
                    def templateFile = "${CFT_TEMPLATE}"
                    print(templateFile)

                    withAWS(region: "${env.AWS_REGION}", credentials: 'my-aws-account') {
                        bat """
                        aws cloudformation validate-template --template-body file://${templateFile}
                        """
                    }
                }
            }
        }

        stage('Show Changeset') {
            when {
                expression { params.ACTION == 'create' || params.ACTION == 'update' }
            }
            steps {
                script {
                    def stackName = getStackName(params.CFT_TEMPLATE)
                    def templateFile = params.CFT_TEMPLATE

                    withAWS(region: "${env.AWS_REGION}", credentials: 'my-aws-account') {
                    env.CHANGESET = bat(
                        script: """
                        @echo off
                        setlocal
                        set AWS_STACK_NAME=${stackName}
                        set AWS_TEMPLATE_FILE=${templateFile}
                        set AWS_CHANGE_SET_TYPE=${params.ACTION.toUpperCase()}
                    
                        aws cloudformation create-change-set ^
                            --stack-name %AWS_STACK_NAME% ^
                            --template-body file://%AWS_TEMPLATE_FILE% ^
                            --change-set-name my-changeset ^
                            --change-set-type %AWS_CHANGE_SET_TYPE% ^
                            --capabilities CAPABILITY_NAMED_IAM ^
                            --query Id ^
                            --output text
                        """,
                        returnStdout: true
                    ).trim()
                    echo "Change Set ID: ${env.CHANGESET}"
                    def describeChangeset = bat(
                        script: """
                        @echo off
                        setlocal
                    
                        aws cloudformation describe-change-set ^
                            --change-set-name ${env.CHANGESET}
                        """,
                        returnStdout: true
                    ).trim() 
                    echo "Changeset Details:\n${describeChangeset}"
                    }
                }
            }
        }

        stage('Approval') {
            when {
                expression { params.ACTION == 'create' || params.ACTION == 'update' }
            }
            steps {
                input message: 'Do you approve the changes?', ok: 'Yes, proceed'
            }
        }

        stage('Execute Action') {
            steps {
                script {
                    def stackName = getStackName(params.CFT_TEMPLATE)
                    def templateFile = params.CFT_TEMPLATE

                    withAWS(region: "${env.AWS_REGION}", credentials: 'my-aws-account') {
                        if (params.ACTION == 'create') {
                            bat(
                                script: """
                                @echo off
                                setlocal
                                aws cloudformation execute-change-set ^
                                    --change-set-name ${env.CHANGESET}
                                """,
                                returnStdout: true
                            )
                        }else if(params.ACTION == 'update'){
                            cfnUpdate(stack: stackName, file: templateFile)
                        }
                        else if (params.ACTION == 'delete') {
                            cfnDelete(stack: stackName)
                        }
                    }
                }
            }
        }
    }

    post {
        always {
            cleanWs()
        }
    }
}

// Function to fetch the list of CloudFormation templates from the checked out repository
def getTemplates() {
    def templates = []
    node {
        // Execute shell command to get a list of .yaml files
        templates = bat(
            script: 'dir /b *.yaml',
            returnStdout: true
        ).trim().split('\n')
    }
    return templates
}


def getStackName(templateFile) {
    // Derive stack name from the template file name
    return templateFile.replace('.yaml', '')
}
