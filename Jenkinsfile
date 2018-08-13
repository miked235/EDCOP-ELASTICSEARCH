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
    /* Master */
    def master_number_scheduled=sh(returnStdout: true, script: "kubectl get sts $user_id-$tool_name-$env.BUILD_ID-$tool_name-master  -o jsonpath={.status.replicas}").trim()
    def master_number_current=sh(returnStdout: true, script: "kubectl get sts $user_id-$tool_name-$env.BUILD_ID-$tool_name-master  -o jsonpath={.status.currentReplicas}").trim()
    def master_number_ready=sh(returnStdout: true, script: "kubectl get sts $user_id-$tool_name-$env.BUILD_ID-$tool_name-master  -o jsonpath={.status.readyReplicas}").trim()
 
    /* Workers */
    def worker_number_scheduled=sh(returnStdout: true, script: "kubectl get sts $user_id-$tool_name-$env.BUILD_ID-$tool_name  -o jsonpath={.status.replicas}").trim()
    def worker_number_current=sh(returnStdout: true, script: "kubectl get sts $user_id-$tool_name-$env.BUILD_ID-$tool_name  -o jsonpath={.status.currentReplicas}").trim()
    def worker_number_ready=sh(returnStdout: true, script: "kubectl get sts $user_id-$tool_name-$env.BUILD_ID-$tool_name  -o jsonpath={.status.readyReplicas}").trim()

    /* Printing Result */
    println("[MASTER] Scheduled Pods: $master_number_scheduled | Ready pods: $master_number_ready | Current pods: $master_number_current")
    println("[WORKER] Scheduled Pods: $worker_number_scheduled | Ready pods: $worker_number_ready | Current Pods: $worker_number_current")

    /* Verifying Result */
    if(master_number_ready==master_number_scheduled) {
      println("Master pods are running")
    } else {
      println("Some or all of the master pods failed")
      error("Some or all of the master pods failed")
    } 
    if (worker_number_scheduled=="0") {
      println("No worker pods scheduled.")
    } else if(worker_number_ready==worker_number_scheduled) {
      println("Worker pods are running")
    } else {
      println("Some or all of the worker pods failed")
      error("Some or all of the worker pods failed")
    }
  }

  stage('Verifying Elasticsearch started on first pods') {
    /* Master */
    def master_command="kubectl get pods  | grep $user_id-$tool_name-$env.BUILD_ID-$tool_name-master | awk "+'{\'print $1\'}'+"| head -1"
    def first_master_pod=sh(returnStdout: true, script: master_command)
    def master_command2="kubectl logs $first_master_pod | grep started"
    println("Master logs:")
    println(master_command2) 
    sh(master_command)

    /* Worker */
    def worker_number_scheduled=sh(returnStdout: true, script: "kubectl get sts $user_id-$tool_name-$env.BUILD_ID-$tool_name  -o jsonpath={.status.replicas}").trim()
    if (worker_number_scheduled!="0") {
      def worker_command="kubectl get pods  | grep $user_id-$tool_name-$env.BUILD_ID-$tool_name | awk "+'{\'print $1\'}'+"| head -1"
      def first_worker_pod=sh(returnStdout: true, script: worker_command)
      def worker_command2="kubectl logs $first_worker_pod | grep started"
      println("Worker logs:")
      println(worker_command2)
      sh(worker_command)
    }
  }

  stage('Verify cluster health') {
    /* Uses master pod */
    def master_command="kubectl get pods  | grep $user_id-$tool_name-$env.BUILD_ID-$tool_name-master | awk "+'{\'print $1\'}'+"| head -1"
    def first_master_pod=sh(returnStdout: true, script: master_command)
    /* You MUST have jq installed on Jenkins' filesystem or container */
    
    
    def health_command="kubectl exec -i " + '$first_master_pod' + " -- bash -c \"curl -X --head data-service" + ':' + "9200/_cluster/health\" | jq --raw-output \'.status\'"
    
    
    println(health_command)
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
    def init_job_command="kubetl get pods | grep $user_id-$tool_name-$env.BUILD_ID-$tool_name-post-installs-job | awk "+'{\'print $1\'}'+"| head -1"
    def init_job_pod=sh(returnStdout: true, script: init_job_command)
    def init_job_status=sh(returnStdout: true, script: "kubectl get pod $init_job_pod -o jsonpath={.status.phase}").trim()
    
    if(init_job_status=="Succeeded") {
      println("Initialization jobs completed successfully.")
    } else {
      println("ERROR: Initialization jobs did not complete sucessfully")
      error("ERROR: Initialization jobs did not complete sucessfully")
    }
  }
}
