def label = "wordsmith-deployment-${UUID.randomUUID().toString()}"
podTemplate(label: label, yaml: """
apiVersion: v1
kind: Pod
spec:
    containers:
    - name: jnlp
    - name: helm
      image: devth/helm
      command:
      - cat
      tty: true
    - name: kubectl
      image: lachlanevenson/k8s-kubectl:v1.10.7
      command:
      - cat
      tty: true
    - name: curl
      image: appropriate/curl
      command:
      - cat
      tty: true
    - name: jdk
      image: openjdk:8-jdk
      command:
      - cat
      tty: true
"""
    ) {
  node (label) {
    def environment
    stage('Load Environment Definition') {
      checkout scm
      environment = readYaml file: 'environment.yaml'
    }
    stage('Update database') {
      container('jdk') {
        dir ('target/wordsmith-db') {
          git "https://github.com/cloudbeers/wordsmith-db.git"

          withEnv(["PG_SQL_JDBC_URL=${environment.database.url}"]) {

            withCredentials([usernamePassword(
             credentialsId: "${environment.database.credentials.jenkinsCredentialsId}", 
             passwordVariable: 'PG_SQL_CREDS_PSW', usernameVariable: 'PG_SQL_CREDS_USR')]) {

              withMaven(mavenOpts: '-Dorg.slf4j.simpleLogger.log.org.apache.maven.cli.transfer.Slf4jMavenTransferListener=warn') {
                  sh "./mvnw validate"

                  for (liquibaseChangeLog in environment.database.liquibaseChangeLogs) {
                    changeLogFile = "src/main/liquibase/changelog-${liquibaseChangeLog}.xml"
                    sh """
                       # Display all changes which will be applied by the Update command
                       ./mvnw liquibase:status -Dliquibase.changeLogFile=${changeLogFile}
                       
                       # Update the database
                       ./mvnw liquibase:update -Dliquibase.changeLogFile=${changeLogFile}
                    """
                    archiveArtifacts artifacts: changeLogFile, fingerprint: true
                  } // for

                  DB_TAG_VERSION = readFile("target/VERSION")
                  sh """
                    # Create a tag in order to rollback if needed
                    ./mvnw liquibase tag:${DB_TAG_VERSION}
                  """
              } // withMaven
            } // withCredentials
          } // withEnvironment
         } // dir
      } // container
    } // stage
    stage('Install Helm Charts') {
        container('helm') {
          sh """
             helm init --client-only
             helm repo add wordsmith https://charts.wordsmith.beescloud.com
             helm repo update
          """
          for (application in environment.applications) {
              def jenkinsCredentials = []
              def arguments = []
              def idx=0
              for (credentials in application.credentials) {
                jenkinsCredentials.add(usernamePassword(credentialsId: "${credentials.jenkinsCredentialsId}", passwordVariable: "CREDS_${idx}_PSW", usernameVariable: "CREDS_${idx}_USR"))
                arguments.add("--set ${credentials.helmUsernameParameter}=\$CREDS_${idx}_USR,${credentials.helmPasswordParameter}=\$CREDS_${idx}_PSW")
              }
              
              if (application.values?.trim()) {
                  arguments.add "--values ${application.values}"
              }
              withCredentials (jenkinsCredentials) {
                try {
                  sh """
                      helm fetch ${application.chart} --version=${application.version}
                      helm upgrade --install ${application.release} ${application.chart} --version=${application.version} --namespace ${environment.namespace} --wait ${arguments.join(' ')}
                  """
                } catch (Exception e) {
                  def deploymentIssue = [fields: [
                               project: [key: 'WOR'],
                               summary: "Deployment failure: ${application.release}",
                               description: "Please go to ${BUILD_URL} and verify the deployment logs",
                               issuetype: [name: 'Bug']]]

                  jiraResponse = jiraNewIssue issue: deploymentIssue
                  echo "https://jira.beescloud.com/projects/WOR/issues/${jiraResponse.data.key}"
                  throw e
                }                
              }
          }
          archiveArtifacts artifacts: "*.tgz", fingerprint: true
          def deploymentIssue = [fields: [
             project: [key: 'WOR'],
             summary: "Verify deployment on ${environment.namespace}",
             description: "Please go to ${BUILD_URL} and verify the deployment logs",
             issuetype: [name: 'Task']]]

          jiraResponse = jiraNewIssue issue: deploymentIssue
          echo "Jira verification task created https://jira.beescloud.com/projects/WOR/issues/${jiraResponse.data.key}"
        } // container
    } // stage
  } // node
} // podTemplate