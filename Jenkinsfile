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
"""
    ) {
  node (label) {
    stage('Install Helm Charts') {
        container('helm') {
          git 'https://github.com/cloudbeers/wordsmith-env-staging.git'
          def environment = readYaml file: 'environment.yaml'
          echo ("$environment")
          for (application in environment.applications) {
              echo "chart ${application.class} $application"

              def jenkinsCredentials = []
              def arguments = []
              def idx=0
              for (credentials in application.credentials) {
                jenkinsCredentials.add(usernamePassword(credentialsId: "${credentials.jenkinsCredentialsId}", passwordVariable: "CREDS_${idx}_PSW", usernameVariable: "CREDS_${idx}_USR"))
                arguments.add("--set ${credentials.helmUsernameParameter}=\$CREDS_${idx}_USR,${credentials.helmPasswordParameter}=\$CREDS_${idx}_PSW")
              }
              
              if (application.values != null && !application.values.empty) {
                  arguments.add "--values ${application.values}"
              }
              withCredentials (jenkinsCredentials) {
                  sh """
                      echo helm fetch ${application.chart} --version=${application.version}
                      echo helm upgrade --install ${application.release} ${application.chart} --version=${application.version} --wait ${arguments.join(' ')}
                  """
                  sh "env"
              }
              archiveArtifacts artifacts: "${application.chart}-${APPLICATION_VERSION}.tgz", fingerprint: true
          }
        } // container
    } // stage
  } // node
} // podTemplate