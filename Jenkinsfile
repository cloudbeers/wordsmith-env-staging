{

    def environment = readYaml("environment.yaml")
    def liquibaseLabel = environment["wordsmith-db"].["liquibaseLabel"]
    def k8sNamespace = environment["kubernetes-namespace"]

	// update DB
	sh """
	    mkdir -p target/wordsmith-db
	    cd target/wordsmith-db
        git clone https://github.com/cloudbeers/wordsmith-db
        ./mvnw liquibase:update -DliquibaseLabel=${liquibaseLabel}
	"""

	// update API
	def wordsmithApiVersion = environment["wordsmith-api"].["version"]

    helm init --client-only
    helm repo add wordsmith http://chartmuseum-chartmuseum.core.svc.cluster.local:8080
    helm repo update

    helm upgrade wordsmith-api-${k8sNamespace} wordsmith/wordsmith-api --version ${wordsmithApiVersion} --install --namespace ${k8sNamespace} --wait \
       --set database.username=${PG_SQL_CREDS_USR},database.password=${PG_SQL_CREDS_PSW}


	// update front
}