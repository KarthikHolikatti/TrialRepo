@Library('jenkins-pipeline-nuget') _
timestamps {

  //###########GLOBALS###############
  def nugetBasePath = 'C:\\Program Files (x86)\\Nuget'
  def nugetPath = "${nugetBasePath}\\nuget.exe"
  def version = "0.0.0"
  def git_url = ''
  def chat_ops_channel = '#jenkins'
  // Artifactory config
  def art_server_id = 'advancedcsg'
  def art_retention_max_builds = 10
  def art_retention_max_days = 183
  def art_target_base_url
  def art_delete_build_artifacts

  if(isMasterOrRelease()) {
     art_target_base_url = 'nuget-release-virtual'
      art_delete_build_artifacts = false
  } else {
      art_target_base_url = 'nuget-snapshot-virtual'
      art_delete_build_artifacts = true
  }

  // The path in Artifactory where the artefacts are uploaded. Version specific part added below
  def art_upload_path = "${art_target_base_url}/ahc/carenotes/components/library-carenotesspineintegration"

  def container_name = "build-tools"
  //###########\GLOBALS##############

  node("aws && windows") {
   ws("workspace/${wsHash.getWsHash(10)}") {

      //docker_snapshots_registry is globally set by jenkins and DOCKER_REGISTRY is used by compose
      env.DOCKER_REGISTRY = env.DOCKER_SNAPSHOT_REGISTRY
      if(isMasterOrRelease()){
          env.DOCKER_REGISTRY = env.DOCKER_RELEASE_REGISTRY
      }
      echo "Docker registry is ${env.DOCKER_REGISTRY}"

      withCredentials([usernamePassword(credentialsId: 'artifactory-ahc-buildmaster', passwordVariable: 'a_password', usernameVariable: 'a_username')]) {

        try{
          timeout(60){
            stage('Checkout'){
              checkout scm

              git_url = bat (script: '@git config remote.origin.url', returnStdout: true).trim()

              //set some variables based on checkout
              version = getVersion(readFile('VERSION.txt'))
              art_upload_path = "${art_upload_path}/${version}/"
              currentBuild.displayName = version
            }

            stage("Compile"){
              def apexbuildtools_image = "${env.DOCKER_RELEASE_REGISTRY}/ahc/apex/apexbuildtools:0.4"
              def auth = "${a_username}:${a_password}".bytes.encodeBase64().toString()

              def build_command = getBuildSteps(version)
              echo "Build command is ${build_command}"

              container_name = "${container_name}-${version}"

              //note tidy up and delete the container in the finally block
              //so it always happens.
              bat """docker login -u ${env.a_username} -p ${env.a_password} ${env.DOCKER_RELEASE_REGISTRY}
              docker run -d --name ${container_name} -v ${env.WORKSPACE}:C:\\src ${apexbuildtools_image} -src_dir "C:\\src" -build_command "${build_command}" > ${pwd()}\\build_container
              docker logs --follow ${container_name}
              mkdir \".\\build\\any cpu\"
              docker cp \"${container_name}:/build-src/AHC.CareNotes.SpineIntegraton.Component.${version}.nupkg\" \"./build/any cpu/AHC.CareNotes.SpineIntegraton.Component.${version}.nupkg\""""

              // get the exit code from the build container to see if failed or not
              build_exit = powershell(returnStdout: true, script: "docker inspect ${container_name} --format='{{.State.ExitCode}}'")

              if(build_exit.toInteger() != 0){
                error("Build failed because docker build container returned non-zero (${build_exit.toInteger()}) exit code.")
              }
            }

            stage('Publish to Artifactory'){

              if (currentBuild.result == null || currentBuild.result == 'SUCCESS') {
                def search = "nuget.version=${version}&repos=${art_target_base_url}"
                def search_url = "https://advancedcsg.jfrog.io/advancedcsg/api/search/prop?${search}"
                echo "Artifactory search query is: ${search_url}"

                try {
                  def stdout = powershell(returnStdout: true, script:  """
                      \$user = "${a_username}"
                      \$password = ConvertTo-SecureString "${a_password}" -AsPlainText -Force
                      \$creds = New-Object System.Management.Automation.PSCredential (\$user, \$password)
                      \$r = Invoke-RestMethod -Uri "${search_url}" -Credential \$creds
                      If (\$r.Results.Length -gt 0) { throw "Results (published artifacts) were found" }
                  """)
                  println stdout

                  echo 'No existing artefacts found'

                  def server = Artifactory.server("${art_server_id}")
                  def buildInfo = Artifactory.newBuildInfo()
                  buildInfo.env.filter.addExclude("*SECRET*")
                  buildInfo.env.filter.addExclude("*SECRET*")
                 buildInfo.env.filter.addExclude("user")
                  buildInfo.env.filter.addExclude("pass")
                  buildInfo.env.capture = true
                  buildInfo.env.collect()

                  def uploadSpec = """{
                  "files": [
                  {
                      "pattern": ".\\build\\any cpu\\*.nupkg",
                      "target": "${art_upload_path}"
                  }]
                  }"""

                  buildInfo.retention maxBuilds: art_retention_max_builds, maxDays: art_retention_max_days, doNotDiscardBuilds: [], deleteBuildArtifacts: art_delete_build_artifacts
                  server.upload(uploadSpec, buildInfo)
                  server.publishBuildInfo buildInfo

                  setStatusOnRepo(git_url, "Happy days", currentBuild.result)
                }
                catch (ex) {
                    echo 'Not pushing to artifactory'

                    echo "Error message: ${ex.getMessage()}"
                    echo "Error string: ${ex.toString()}"

                    currentBuild.result = 'FAILURE'
                    setStatusOnRepo(git_url, "Unhappy days", currentBuild.result)
                }
              }

            }
          }
        } catch(e){

          currentBuild.result = 'FAILED'
          slackAlert.SendMessage("Error building ${env.JOB_NAME} - ${e.getMessage()} (${env.BUILD_URL})", 'danger', chat_ops_channel, 'ahcindigo', 'ahcindigo-slack-token')
          throw e

        } finally{

          slackAlert.Send(currentBuild, chat_ops_channel, 'ahcindigo', 'ahcindigo-slack-token')
          stage('Clean Up'){

            try{
              bat "docker rm -f ${container_name}"
            } catch(e){
              slackAlert.SendMessage("Error building ${env.JOB_NAME} - ${e.getMessage()} (${env.BUILD_URL})", 'danger', chat_ops_channel, 'ahcindigo', 'ahcindigo-slack-token')
            }

            advancedNuget.cleanCache(nugetPath)
          }

        }
      }
    }
  }
}

def getVersion(text) {
  def matcher = text =~ '^BUILD_VERSION = "(.*?)"'
  revision = ""
  if(!isMasterOrRelease()) {
    branch_name = env.BRANCH_NAME
    if(branch_name.length() > 20){
      branch_name = branch_name.substring(0,19)
    }
    revision = ".${env.BUILD_NUMBER}-${branch_name}".toLowerCase()
  }
                matcher ? "${matcher[0][1]}${revision}" : null
}

def isMasterOrRelease(){
  return env.BRANCH_NAME == 'master' || env.BRANCH_NAME.endsWith('/master') || env.BRANCH_NAME.startsWith('release')
}

def getBuildSteps(version){
  return "C:\\nuget\\nuget.exe sources Add -Name ahcdev -Source http://192.168.183.42/nuget; C:\\nuget\\nuget.exe restore .\\src\\library-carenotesspineintegration.sln; msbuild /p:Configuration=Release /t:build /verbosity:detailed .\\src\\library-carenotesspineintegration.sln; if(\$LASTEXITCODE -ne 0){ echo 'non-zero exit code from msbuild, exiting'; exit \$LASTEXITCODE } else { echo 'Build exit code was 0, continuing.' }; pushd .\\src\\Spine Integration Component; echo 'running BuildComponent.ps1'; .\\BuildComponent.ps1; popd; C:\\nuget\\nuget.exe pack -Version ${version} .\\src\\packaging\\nuspec\\ahc.carenotes.spineintegration.component.nuspec;"
}

def setStatusOnRepo(String gitPath = '', String buildMessage = 'Happy days', String buildStatus = 'SUCCESS')
{
    buildStatus = buildStatus ?: 'SUCCESS'
    echo "Setting git status to ${buildStatus}"

    def gitCommit = bat (
        script: '@git rev-parse HEAD',
        returnStdout: true
    ).trim()

    step([$class: 'GitHubCommitStatusSetter', commitShaSource: [$class: 'ManuallyEnteredShaSource', sha: "${gitCommit}"], reposSource: [$class: 'ManuallyEnteredRepositorySource', url: "${gitPath}"], statusResultSource: [$class: 'ConditionalStatusResultSource', results: [[$class: 'AnyBuildResult', message: "${buildMessage}", state: "${buildStatus}"]]]])
}
