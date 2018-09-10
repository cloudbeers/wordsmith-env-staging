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
          sh """
             helm init --client-only
             helm repo add wordsmith https://charts.wordsmith.beescloud.com
             helm repo update
          """
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
              
              if (application['values'] != null && ! application['values'].trim().isEmpty()) {
                  arguments.add "--values ${application.values}"
              } else {
                  echo "${application.release} has NO values"
              }
              withCredentials (jenkinsCredentials) {
                  sh """
                      helm fetch ${application.chart} --version=${application.version}
                      helm upgrade --install ${application.release} ${application.chart} --version=${application.version} --wait ${arguments.join(' ')}
                  """
                  sh "env"
              }
          }
          archiveArtifacts artifacts: "*.tgz", fingerprint: true
        } // container
    } // stage
  } // node
} // podTemplate