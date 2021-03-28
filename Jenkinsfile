@Library('pipeline-library') _
node ('master'){
try {
  stage('Checking out') {
    git branch: 'main', changelog: false, credentialsId: 'git-andreyess-private-key', poll: false, url: 'https://github.com/andreyess/build-tools.git'
  }
}
catch (err) {
    echo "Caught: ${err}"
    currentBuild.result = 'FAILURE'
    //step([$class: 'Mailer', recipients: 'andrey.karpyza.steam@gmail.com'])
}
try {
    stage ('Building code') {
    mvnHome = tool name: 'MAVEN', type: 'maven'
    sh "'${mvnHome}/bin/mvn' -Dmaven.test.failure.ignore clean package -f helloworld-project/helloworld-ws/pom.xml"
    }
}
catch (err) {
    echo "Caught: ${err}"
    currentBuild.result = 'FAILURE'
    //step([$class: 'Mailer', recipients: 'andrey.karpyza.steam@gmail.com'])
}
try {
    stage ('Sonar scan') {
        withSonarQubeEnv(credentialsId: 'sonar-secret') {
            sonarHome = tool name: 'SONAR', type: 'hudson.plugins.sonar.SonarRunnerInstallation'
            sh "${sonarHome}/bin/sonar-scanner -Dsonar.java.binaries=helloworld-project/helloworld-ws/target/classes -Dsonar.host.url=http://sonar.akarpyza.lab.playpit.by/ -Dsonar.projectKey=MNT:akarpyza-pipeline -Dsonar.sources=helloworld-project/helloworld-ws -Dsonar.projectName=\"MNT 11 akarpyza pipeline\" -Dsonar.projectVersion=${BUILD_TAG}"
        }
    }
}
catch (err) {
    echo "Caught: ${err}"
    currentBuild.result = 'FAILURE'
    //step([$class: 'Mailer', recipients: 'andrey.karpyza.steam@gmail.com'])
}
try {
    stage('Testing') {
        mvnHome = tool name: 'MAVEN', type: 'maven'
        parallel (
            "pre-integration-test": {
                sh "'${mvnHome}/bin/mvn' test -f helloworld-project/helloworld-ws/pom.xml"
            }, "integration-test": {
                echo 'sh "${mvnHome}/bin/mvn" test -f helloworld-project/helloworld-ws/pom.xml'
            }, "post-integration-test": {
                echo 'sh "${mvnHome}/bin/mvn" test -f helloworld-project/helloworld-ws/pom.xml'
            },
            failFast: true
        )
    }
}
catch (err) {
    echo "Caught: ${err}"
    currentBuild.result = 'FAILURE'
    //step([$class: 'Mailer', recipients: 'andrey.karpyza.steam@gmail.com'])
}
try {
    stage('Triggering job and fetching') {
      build propagate: false, job: 'MNTLAB-akarpyza-child1-build-job', parameters: [string(name: 'BRANCH_NAME', value: 'akarpyza')]
      copyArtifacts fingerprintArtifacts: true, projectName: 'MNTLAB-akarpyza-child1-build-job', selector: lastSuccessful()
    }
}
catch (err) {
    echo "Caught: ${err}"
    currentBuild.result = 'FAILURE'
   // step([$class: 'Mailer', recipients: 'andrey.karpyza.steam@gmail.com'])
}
try {
stage('Packaging and Publishing results') {
    parallel(
       "Push artifact": {
            sh "tar -cvf pipeline-akarpyza-${BUILD_NUMBER}.tar.gz helloworld-project/helloworld-ws/target/helloworld-ws.war output.txt"

            publishing.BuildAndPublishToNexus("${BUILD_NUMBER}", 'nexus.akarpyza.lab.playpit.by/repository/artifacts_hosted/', 'raw-repo', '7.1.0.GA', 'org', 'nexus creds')
        },
        "Push image": {
            sh 'echo "FROM jboss/wildfly" >> Dockerfile'
            sh 'echo "EXPOSE 8080" >> Dockerfile'
            sh 'echo "ADD helloworld-project/helloworld-ws/target/helloworld-ws.war /opt/jboss/wildfly/standalone/deployments/" >> Dockerfile'

            publishing.PushToDocker("${BUILD_NUMBER}", 'nexus creds', 'https://docker.akarpyza.lab.playpit.by', 'DOCKER')
        }
    )
}
}
catch (err) {
    echo "Caught: ${err}"
    currentBuild.result = 'FAILURE'
    //step([$class: 'Mailer', recipients: 'andrey.karpyza.steam@gmail.com'])
}
try {
    stage('Asking for manual approval') {
        timeout(time: 120, unit: 'SECONDS') {
            input 'It\'s successfully builded. Please, approve changes'
        }
    }
}
catch (err) {
    echo "Caught: ${err}"
    currentBuild.result = 'FAILURE'
    //step([$class: 'Mailer', recipients: 'andrey.karpyza.steam@gmail.com'])
}
try {
    stage('Deployment') {
        sh """echo "apiVersion: apps/v1
kind: Deployment
metadata:
  name: helloworld-ws-deployment
  namespace: jenkins
spec:
  selector:
    matchLabels:
      app: helloworld-ws
  minReadySeconds: 5
  template:
    metadata:
      labels:
        app: helloworld-ws
    spec:
      containers:
      - name: tomcat-helloworld-ws
        image: docker.akarpyza.lab.playpit.by/helloworld-akarpyza:$BUILD_NUMBER
        ports:
        - containerPort: 8080
        readinessProbe:
          httpGet:
            path: /helloworld-ws
            port: 8080
          periodSeconds: 10
          timeoutSeconds: 5
          successThreshold: 2
          failureThreshold: 5
      imagePullSecrets:
      - name: nexus
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 2
      maxUnavailable: 0" > deployment.yml"""
        step([$class: 'KubernetesEngineBuilder', projectId: "splendid-sunset-291720", clusterName: "cluster-2", zone: "us-west1-a", manifestPattern: 'deployment.yml', credentialsId: "splendid-sunset-291720"])
    }
}
catch (err) {
    echo "Caught: ${err}"
    currentBuild.result = 'FAILURE'
    //step([$class: 'Mailer', recipients: 'andrey.karpyza.steam@gmail.com'])
}
finally {
        stage('Sending status') {
            currentBuild.result = 'SUCCESS'
            //step([$class: 'Mailer', recipients: 'andrey.karpyza.steam@gmail.com'])
        }
    }
}
