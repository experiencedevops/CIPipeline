/* Secrets Configuration */
scmcredId = 'GitHub'
nexuscredId = 'nexus'
jiracredId = 'JIRA'

/* Application Constants */
sonarQubeURL='http://34.239.251.172:9000'
bbprotocol='https'
bbURL='github.com'
appRepo='github.com/experiencedevops/customerservice/'
automationRepo='github.com/experiencedevops/CIPipeline.git'
automationRepoDirectory='automation'
nexusBase = 'http://34.200.232.169:8081/nexus/content/repositories'
nexus_rest_3='http://34.200.232.169:8081/service/siesta/rest/v1/script'
nexus_rest='http://34.200.232.169:8081/nexus/service/local/artifact/maven/content'
relbranch_devops='master'
relbranch_config='master'
to_emailid='experiencedigitial@gmail.com'

node(){

   // Mark the code checkout 'stage'....
   def mvnHome = tool 'M3'
   def commit_id
   
   stage 'Checkout'
   deleteDir() 
   try {
       checkoutscm()
       currentBuild.displayName="#${env.BUILD_NUMBER}"
       currentBuild.result='SUCCESS' 
   } catch (e) {
      currentBuild.result='FAILED'
      sendEmail( 'FAILED' )
      throw e
   }

  stage 'Build application and Run Unit Test'
   try {
      
      if (isUnix()){
         sh "${mvnHome}/bin/mvn -Dmaven.test.failure.ignore clean package"
      } else {
         bat("${mvnHome}/bin/mvn -Dmaven.test.failure.ignore clean package")
      }
   } catch (e) {
      currentBuild.result='FAILED'
      sendEmail( 'FAILED' )
      throw e
   }
   
   stage 'Run SonarQube Analysis'
   try {
     if (isUnix()) { 
         /*sh "${mvnHome}/bin/mvn clean org.jacoco:jacoco-maven-plugin:prepare-agent test"*/
         sh "${mvnHome}/bin/mvn package sonar:sonar -Dsonar.host.url='${sonarQubeURL}'"
     }else{
         /*bat("${mvnHome}/bin/mvn clean org.jacoco:jacoco-maven-plugin:prepare-agent test")*/
         bat("${mvnHome}/bin/mvn package sonar:sonar -Dsonar.host.url='${sonarQubeURL}'")
     }
   } catch (e){
      currentBuild.result='FAILED'
      sendEmail( 'FAILED' )
      throw e
   }
   
   stage ('Deploy to Nexus'){
   try {
      
      // install Maven and add it to the path
      env.PATH = "${tool 'M3'}/bin:${env.PATH}"
      configFileProvider([configFile(fileId: 'global-config', variable: 'MAVEN_SETTINGS')]) {
              sh "mvn help:effective-settings"
              sh 'mvn -s $MAVEN_SETTINGS deploy'
      }
   } catch(e){
      currentBuild.result='FAILED'
      sendEmail( 'FAILED' )
      throw e
   }
}
 
 stage('docker build/push') {
    docker.withRegistry('https://index.docker.io/v1/', 'GitHub') {
       def app = docker.build("experiencedevops/customerservice:latest", '.').push()
     }
   }
   
stage 'Send Mail' 
      try {
      sendEmail( 'SUCCESS' )
    } catch (e) {
         currentBuild.result='FAILED'
         throw e
     }
}

 /*stage("SonarQube Quality Gate") { 
        timeout(time: 1, unit: 'HOURS') { 
           def qg = waitForQualityGate() 
           if (qg.status != 'OK') {
             error "Pipeline aborted due to quality gate failure: ${qg.status}"
           }
        }
    }*/

def checkoutscm() {      
   withCredentials([[$class: 'UsernamePasswordMultiBinding', credentialsId: "${scmcredId}", usernameVariable: 'bb_userid', passwordVariable: 'bb_password']]) {    
      dir ("${WORKSPACE}") {  	  	      
              git credentialsId: "${scmcredId}", poll: false, url: "${bbprotocol}://${env.bb_userid}:${env.bb_password}@${appRepo}", branch: "${relbranch_config}"
              sh "git rev-parse --short HEAD > .git/commit-id"                        
              commit_id = readFile('.git/commit-id').trim()
              println commit_id
       }   
   }
   
  withCredentials([[$class: 'UsernamePasswordMultiBinding', credentialsId: "${scmcredId}", usernameVariable: 'bb_userid', passwordVariable: 'bb_password']]) {    
     dir ("${WORKSPACE}/${automationRepoDirectory}") {  	  	      
              git credentialsId: "${scmcredId}", poll: false, url: "${bbprotocol}://${env.bb_userid}:${env.bb_password}@${automationRepo}", branch: "${relbranch_config}"                       
       }   
   }
}

def sendEmail( Status ) {        
    if ( "${Status}" == 'SUCCESS' || "${Status}" == 'UNSTABLE' )
    {        
        /*emailbody = readFile 'builddesc.txt'*/  
        emailbody   = 'Build Successful. \n Docker Image has been pushed to experiencedevops/customerservice repository'
        currentBuild.result = "${Status}"
    }  
    else if ( "${Status}" == 'FAILED' )
    {
        emailbody = 'Deployment Failed !!! Please check attached logs.'        
        currentBuild.result = 'FAILED'
    }
    emailext attachLog: true, body: "${emailbody}", compressLog: true, subject: "Build #${env.BUILD_NUMBER} - Deployment ${Status}.", to: "${to_emailid}"
}

def uploadArtifacts() { 
    withCredentials([[$class: 'UsernamePasswordMultiBinding', credentialsId: "${nexuscredId}", usernameVariable: 'nexus_userid', passwordVariable: 'nexus_password']]) {
       sh "chmod ugo+rwx ${WORKSPACE}/${automationRepoDirectory}/nexusDeploy.sh"
       sh "${WORKSPACE}/${automationRepoDirectory}/nexusDeploy.sh ${WORKSPACE} ${nexus_rest} ${nexus_userid} ${nexus_password} ${BUILD_NUMBER}"
    }
    
}
