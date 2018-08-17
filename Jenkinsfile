#!/usr/bin/groovy

@Library('github.com/lachie83/jenkins-pipeline@dev')
def pipeline = new io.estrado.Pipeline()


node {
  def app



  def pwd = pwd()
  def tool_name="elasticsearch"
  def support_tool_name="curator"
  def container_dir = "$pwd/containers/"
  def custom_image = "images.elasticsearch"
  /* #def custom_values_url = "http://repos.sealingtech.com/cisco-c240-m5/elasticsearch/values.yaml" */
  def user_id = ''
  wrap([$class: 'BuildUser']) {
      echo "userId=${BUILD_USER_ID},fullName=${BUILD_USER},email=${BUILD_USER_EMAIL}"
      user_id = "${BUILD_USER_ID}"
  }

  sh "env"

  def container_tag = "gcr.io/edcop-dev/$user_id-$tool_name"
  def support_container_tag = "gcr.io/edcop-dev/$user_id-$support_tool_name"

  stage('Clone repository') {
      /* Let's make sure we have the repository cloned to our workspace */
      checkout scm
  }

  stage('helm lint') {
      sh "helm lint $tool_name"
  }

  stage('helm deploy') {
      sh "helm install --name='$user_id-$tool_name-$env.BUILD_ID' $tool_name"
  }

  stage('sleeping 3 minutes') {
    sleep(180)
  }

  stage('Verifying running pods') {
    /* Default Nodes */
    def number_scheduled=sh(returnStdout: true, script: "kubectl get sts $user_id-$tool_name-$env.BUILD_ID-$tool_name  -o jsonpath={.status.replicas}").trim()
    def number_current=sh(returnStdout: true, script: "kubectl get sts $user_id-$tool_name-$env.BUILD_ID-$tool_name  -o jsonpath={.status.currentReplicas}").trim()
    def number_ready=sh(returnStdout: true, script: "kubectl get sts $user_id-$tool_name-$env.BUILD_ID-$tool_name  -o jsonpath={.status.readyReplicas}").trim()

    /* Printing Result */
    println("Scheduled Pods: $number_scheduled | Ready pods: $number_ready | Current pods: $number_current")

    /* Verifying Result */
    if(number_current==number_scheduled) {
      println("All pods are running")
    } else {
      println("Some or all of the pods failed")
      error("Some or all of the pods failed")
    } 
  }

  stage('Verifying Elasticsearch started on first pods') {
    /* Default Nodes */
    def command="kubectl get pods | grep $user_id-$tool_name-$env.BUILD_ID-$tool_name | awk "+'{\'print $1\'}'+"| head -1"
    def first_pod=sh(returnStdout: true, script: command)
    def command2="kubectl logs $first_pod | grep started"
    println("Elasticsearch logs:")
    println(command2) 
    sh(command)
  }

  stage('Verify cluster health') {
    def command="kubectl get pods | grep $user_id-$tool_name-$env.BUILD_ID-$tool_name | awk "+'{\'print $1\'}'+"| head -1"
    def first_pod=sh(returnStdout: true, script: command).trim()
    /* You MUST have jq installed on Jenkins' filesystem or container */
    def health_command="kubectl exec -i " + "$first_pod" + " -- bash -c \"curl -X --head data-service" + ':' + "9200/_cluster/health\" | jq --raw-output \'.status\'"
    def health=sh(returnStdout: true, script: health_command).trim()

    /* Health should be green */
    if(health=="green") {
      println("Cluster health is green")
    } else if (health=="yellow") {
      println("WARNING: Cluster health is yellow")
    } else {
      println("ERROR: Cluster health is red, something went wrong")
      error("ERROR: Cluster health is red, something went wrong")
    }
  }
  
  stage('Verify init scripts completed') {
    /* Get elasticsearch template init logs */
    def init_job_command="kubectl get pods | grep $user_id-$tool_name-$env.BUILD_ID-$tool_name-post-installs-job | awk "+'{\'print $1\'}'+"| head -1"
    def init_job_pod=sh(returnStdout: true, script: init_job_command).trim()
    def init_job_status=sh(returnStdout: true, script: "kubectl get pod $init_job_pod -o jsonpath={.status.phase}").trim()
    
    if(init_job_status=="Succeeded") {
      println("Initialization jobs completed successfully.")
    } else {
      println("ERROR: Initialization jobs did not complete sucessfully")
      error("ERROR: Initialization jobs did not complete sucessfully")
    }
  }
}
