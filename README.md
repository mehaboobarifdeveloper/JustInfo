pipeline {
    agent any
    environment {
        JIRA_BASE_URL = 'https://your-jira-instance.atlassian.net'
        JIRA_USERNAME = 'your-email@example.com'
        JIRA_API_TOKEN = 'your-api-token'
        XRAY_CLIENT_ID = 'your-xray-client-id'
        XRAY_CLIENT_SECRET = 'your-xray-client-secret'
        TEST_PLAN_KEY = 'TP-123' // Your Test Plan key
        PROJECT_KEY = 'YOUR_PROJECT_KEY' // Your Jira project key
    }
    stages {
        stage('Build and Test') {
            steps {
                // Your build and test steps, generating the Cucumber JSON report
                sh './gradlew test'
            }
        }
        stage('Upload Cucumber JSON to Xray') {
            steps {
                script {
                    // Obtain Xray authentication token
                    def response = httpRequest(
                        httpMode: 'POST',
                        url: "${env.JIRA_BASE_URL}/api/v2/authenticate",
                        requestBody: "{\"client_id\": \"${env.XRAY_CLIENT_ID}\", \"client_secret\": \"${env.XRAY_CLIENT_SECRET}\"}",
                        contentType: 'APPLICATION_JSON'
                    )
                    def authToken = response.content

                    // Create Test Execution
                    def testExecutionPayload = [
                        fields: [
                            project: [
                                key: "${env.PROJECT_KEY}"
                            ],
                            summary: 'Automated Test Execution',
                            issuetype: [
                                name: 'Test Execution'
                            ],
                            description: 'Automated Test Execution created by Jenkins'
                        ]
                    ]
                    response = httpRequest(
                        httpMode: 'POST',
                        url: "${env.JIRA_BASE_URL}/rest/api/2/issue",
                        requestBody: JsonOutput.toJson(testExecutionPayload),
                        customHeaders: [[name: 'Authorization', value: "Bearer ${authToken}"]],
                        contentType: 'APPLICATION_JSON'
                    )
                    def testExecutionKey = new groovy.json.JsonSlurper().parseText(response.content).key

                    // Upload the Cucumber JSON report to the Test Execution
                    def cucumberReport = readFile file: 'build/reports/cucumber.json'
                    httpRequest(
                        httpMode: 'POST',
                        url: "${env.JIRA_BASE_URL}/api/v2/import/execution/cucumber?testExecIssueKey=${testExecutionKey}",
                        requestBody: cucumberReport,
                        customHeaders: [[name: 'Authorization', value: "Bearer ${authToken}"]],
                        contentType: 'APPLICATION_JSON'
                    )

                    // Associate Test Execution with Test Plan
                    def associatePayload = [
                        testExecIssueKey: testExecutionKey,
                        testPlanKey     : env.TEST_PLAN_KEY
                    ]
                    httpRequest(
                        httpMode: 'POST',
                        url: "${env.JIRA_BASE_URL}/api/v2/testplan/addtests",
                        requestBody: JsonOutput.toJson(associatePayload),
                        customHeaders: [[name: 'Authorization', value: "Bearer ${authToken}"]],
                        contentType: 'APPLICATION_JSON'
                    )
                }
            }
        }
    }
}
