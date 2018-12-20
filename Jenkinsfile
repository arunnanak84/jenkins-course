node {
          git branch: "${env.GIT_BRANCH}",
          credentialsId: 'e17359e0-1721-44a1-bc51-327c6c3d639c',
          recursiveSubmodules: true,
          url: 'https://github.com/arunnanak84/jenkins-course.git'
          
          stage('Add Job Description') {
              dp = "Branch: ${env.GIT_BRANCH}"
              currentBuild.description = dp
          }

         stage("Unit test") {
            sh"""mvn clean install && mvn test"""
          }
        
             
       stage("Build or Compile") {
        sh "make all"
      }

       stage('Code Quality with SonarQube') {
                                // requires SonarQube Scanner 2.8+
                    // Value of this home is
                    // /var/jenkins_home/tools/hudson.plugins.sonar.SonarRunnerInstallation/sonar
                    // As in Jenkins Global Tool Config we have specified name
                    // as "sonar"
                    // scannerHome = tool 'sonar'
                    // echo scannerHome
        sh 'cd $WORKSPACE/source/plugins/quick-joiner && mvn sonar:sonar  -Dsonar.host.url=http://IP_address:9000  -Dsonar.login=ldjsjfksjdfklsjdklfjskdjf'
        }

stage(' Build RPM ') {
    def branch = "${env.GIT_BRANCH}".replace('refs/heads/', '')
            def buildType = branch.split('/').first()
            def VER = readFile ".VERSION"
            def VERSION = VER.replaceAll("[\r\n]+","")
            def REL = sh(
                  script: 'git rev-parse --short HEAD',
                  returnStdout: true)
            def RELEASE = REL.replaceAll("[\r\n]+","")
            def finalrelease = currentBuild.getNumber()
            def masterDockerTag = VERSION + "-" + RELEASE
            def releaseDockerTag =  VERSION + "-" + finalrelease
           sh"""
           chmod +x rpm-mgmt/build_rpm.sh
           """
        if (branch == 'master' || env.SMOKE_TEST == "YES") {
          sh"""
          rpm-mgmt/build_rpm.sh ${VERSION} guavus_${RELEASE}
          """
          }

      	if (buildType == 'release') {
              sh"""
               rpm-mgmt/build_rpm.sh ${VERSION} ${finalrelease}
               """
            }
        }

    stage('RPM push') {
          def branch = "${env.GIT_BRANCH}".replace('refs/heads/', '')
          def VER = readFile ".VERSION"
          def buildType = branch.split('/').first()
          def VERSION = VER.replaceAll("[\r\n]+","")
          def server = Artifactory.server('ggn-artifactory')

          if (branch == 'master' || env.SMOKE_TEST == "YES") {
             // Create the upload spec for DEV.
    def uploadSpecDev = """{
        "files": [
                {
                    "pattern": "dist/(*).rpm",
                    "target": "ggn-dev-rpms/cdap-plugins/${VERSION}/dev/",
                    "props": "BRANCH=${branch};VERSION=${VERSION}"
                }
            ]
        }"""
    def buildInfoDev = server.upload spec: uploadSpecDev
    server.publishBuildInfo buildInfoDev
        }
         if (buildType == 'release') {
             
             // Create the upload spec for Release.
    def uploadSpecRel = """{
        "files": [
                {
                    "pattern": "dist/(*).rpm",
                    "target": "ggn-dev-rpms/cdap-plugins/${VERSION}/release/",
                    "props": "BRANCH=${branch};VERSION=${VERSION}"
                }
            ]
        }"""
             def buildInfoRel = server.upload spec: uploadSpecRel
          server.publishBuildInfo buildInfoRel
            }
            // Upload to Artifactory.
      }
} 
