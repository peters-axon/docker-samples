import javax.management.remote.JMXServiceURL as JmxUrl
import javax.management.remote.JMXConnectorFactory as JmxFactory
import javax.management.remote.JMXConnector

pipeline {
  agent {
    label 'docker-compose'
  }

  options {
    disableConcurrentBuilds()
  }

  triggers {
    cron '@midnight'
  }

  stages {
    stage('pull-engine') {
      steps {
        pullEngineImage()             
      }
    }

    stage('test') {
      steps {
        script {
          def examples = [
            // 'ivy': { assertIvyIsRunningInDemoMode() },
            // 'ivy-db-postgres': { assertIvyIsNotRunningInDemoMode() },
            // 'ivy-db-mysql': { assertIvyIsNotRunningInDemoMode() },
            // 'ivy-db-mariadb': { assertIvyIsNotRunningInDemoMode() },
            // 'ivy-db-mssql': { assertIvyIsNotRunningInDemoMode() },
            // 'ivy-deploy-app': { assertAppIsDeployed("app") },

            // 'ivy-environment-variables': { assertIvyIsNotRunningInDemoMode() },
            // 'ivy-logging': { assertIvyConsoleLog("ivy-logging", "Loaded configurations of 'file:/opt/ivy/configuration/ivy.yaml[prefixed: ivy.]'") },
            // 'ivy-openldap': { assertLogin("ldap", "rwei", "rwei") },
            // 'ivy-secrets': { assertIvyIsNotRunningInDemoMode() },

            
            //'ivy-visualvm': { assertJmxConnection() },
            'ivy-elasticsearch': { assertBusinessData() },  
          ]

          
          //export VISUAL_VM_EXAMPLE_REMOTE_HOST_IP=

          examples.each { entry ->
            def example = entry.key;
            def assertion = entry.value;

            echo "START TESTING $example"
            try {
              dockerComposeUp(example)
              waitUntilIvyIsRunning()
              assertion.call()
            } finally {
              dockerComposeDown(example)
            }
          }

          //unset VISUAL_VM_EXAMPLE_REMOTE_HOST_IP
        }
      }
    }

    stage('archive') {
      steps {        
        archiveArtifacts allowEmptyArchive: true, artifacts: 'warn.log'
      }
    }
  }
}

def pullEngineImage() {
  sh 'docker pull axonivy/axonivy-engine:dev'
}

def dockerComposeUp(folder) {
  sh "docker-compose -f $folder/docker-compose.yml up -d"
}

def dockerComposeDown(folder) {
  sh "docker-compose -f $folder/docker-compose.yml down"
}

def waitUntilIvyIsRunning() {
  timeout(2) {
    waitUntil {
      def r = sh script: 'wget -q http://localhost:8080/ivy -O /dev/null', returnStatus: true
      return (r == 0);
    }
  }
}

def assertIvyIsRunningInDemoMode() {
  def responseHtml = sh (script: "wget -qO- http://localhost:8080/ivy/info/index.jsp", returnStdout: true)
  if (!responseHtml.contains('Demo Mode')) {      
    writeWarnLog('ivy is not running in demo mode')
  }
}

def assertIvyIsNotRunningInDemoMode() {
  def responseHtml = sh (script: "wget -qO- http://localhost:8080/ivy/info/index.jsp", returnStdout: true)
  if (responseHtml.contains('Demo Mode')) {      
    writeWarnLog('ivy is running in demo mode')
  }
}

def assertAppIsDeployed(applicationName) {
  def responseHtml = sh (script: "wget -qO- http://localhost:8080/ivy/wf/$applicationName/applicationHome", returnStdout: true)
  if (!responseHtml.contains("This is the home of the application $applicationName")) {
    writeWarnLog("application $applicationName is not deployed")
  }
}

def assertIvyConsoleLog(folder, message) {
  def log = sh (script: "docker-compose -f $folder/docker-compose.yml logs", returnStdout: true)
  if (!log.contains(message)) {
    writeWarnLog("console log of ivy does not contain $message")
  }
}

def assertLogin(application, user, password) {
  def htmlResponse = sh (script: "curl 'http://localhost:8080/ivy/wf/$application/login.jsp' --data 'username=$user&password=$password' -L -i -b cookie.txt", returnStdout: true)
  if (!htmlResponse.contains("Logged in as $user")) {
    writeWarnLog("could not login to application $application as $user")
  }
}

def assertBusinessData() {
  // 1. Deploy Test Project
  sh "docker cp test.iar ivy-elasticsearch_ivy_1:/opt/ivy/deploy/test.zip"  
  sleep(5) // wait until is deployed

  // 2. Execute Process which create business data
  sh "curl 'http://localhost:8080/ivy/pro/test/test/1665799EBA281E4C/start.ivp' --user elastic:changeme"

  // 3. Query Elastic Search
  input 'hey'
  def response = sh (script: "wget -qO- http://localhost:9200/_cat/indices", returnStdout: true)
  def elasticSearchIndex = "ivy.businessdata-test.testbusinessdata";  
  echo "elastic search response: $response"
  if (!response.contains(elasticSearchIndex)) {
    writeWarnLog("could not find elastic search index $elasticSearchIndex in response $response")
  }
}

def assertJmxConnection() {
  def creds = ['admin', 'admin'] as String[]
  def env = [ (JMXConnector.CREDENTIALS) : creds ]
  //def serverUrl = 'service:jmx:rmi:///jndi/rmi://192.168.3.136:9003/jmxrmi'
  def serverUrl = 'service:jmx:rmi:///jndi/rmi://localhost:9003/jmxrmi'
  String beanName = "ivy Engine:type=Application,name=System"
  echo "try to connect $serverUrl with env $env to bean $beanName"

  def server = JmxFactory.connect(new JmxUrl(serverUrl), env).MBeanServerConnection
  def dataSystem = new GroovyMBean(server, beanName)
  echo "jmx connected to:\n$dataSystem\n"

  def systemAttribute = dataSystem.getProperty("system");
  if (systemAttribute == false) {
    writeWarnLog("jmx attribute 'system' must be set to true of the system application, but was $systemAttribute")
  }
}

def writeWarnLog(message) {
  currentBuild.result = 'UNSTABLE'
  sh "echo $message >> warn.log"
}
