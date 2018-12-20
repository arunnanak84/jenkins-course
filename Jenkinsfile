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


---------------
          
          
          pipeline {
  agent any
    //agent { label 'slave' }
	environment {
    // Define global environment variables in this section
    buildNum = currentBuild.getNumber()
    buildType = BRANCH_NAME.split('/').first()
    branchVersion = BRANCH_NAME.split('/').last()
    buildVersion = '0.0.1'
  }
  stages {
    stage("Compile & Code Quality") {
    parallel {
    stage("Compile") {
      steps {
        echo "Run Commmands to compile code"
        sh "mvn compile"
      }
    }
    stage("Unit test") {
      steps {
        echo "Running Unit Tests..."
        sh "#mvn test"
      }
    }
    stage("Code coverage") {
      steps {
        echo "Running Code Coverage..."
        sh "mvn cobertura:cobertura -Dcobertura.report.format=xml"
      }
    }
    stage("Code Quality") {
      steps {
        echo "Run Commmands to execute code quality test"
        sh "mvn sonar:sonar  -Dsonar.host.url=http://192.168.135.114:9000  -Dsonar.login=4e2573327e1f23649e58cca3b689f9cab19ef751"
      }
    }
    stage("Static code analysis") {
      steps {
        echo "Run Commmands to execute static code analysis test"
        sh "mvn findbugs:findbugs"
       }
      }
     }
    }
    stage("Build") {
      steps {
        echo "Building..."
        sh "mkdir -p build;cd build;cmake ..;make interactiveservice_build; cd ../"
      }
    }
    stage("Build rpm") {
      steps {
        echo "Building..."
        sh "cd build;make interactiveservice_rpm; cd ../"
      }
    }
    stage("push rpm") {
      steps {
        echo "Building..."
        sh "cd build;make raf_push_rpms; cd ../"
      }
    }

    stage("Clean Previous Docker Images") {
        steps {
            echo "Removing previous docker images..."
            sh "make clean"
        }
    }

    stage('Create Docker Image and push to artifactory') {
      steps {
        echo "Creating docker build..."
        sh "cd build; make interactiveservice_docker; cd ../"
      }
    }

    stage("Deploy on feature/fix/PR ephemeral test environments") {
      when {
        expression {
          env.buildType ==~ /(feature|PR-.*|fix)/
        }
      }
      steps {
        // Stubs should be used to perform functional testing
        echo "Deploy the Artifact on ephemeral environment"
        echo "Trigger test run to verify code changes"
      }
    }
    stage("Deploy and test on the QA environment") {
      when {
        expression {
          // Run only for buildTypes master or Release
          env.buildType in ['master','release']
        }
      }
      steps {
        echo "Deploy the Artifact on QA setup"
        //sh "virtualenv venv && source venv/bin/activate && cd automation && pip install -r requirements.txt --extra-index-url http://192.168.192.201:5050/simple/ --trusted-host 192.168.192.201 && python --version && python -m pytest -k=test_interactiveservice_installer --testbed=resources/testbeds/security004.yml --solutionInstallerConfigFile=resources/installer/solution_installer_config_security004.yml tests/solution_reinstaller.py && deactivate"
      }
    }

  }
  post {
    success {
      echo 'Build Success'
    }

    unstable {
      echo 'Test Cases Failed'
    }
    failure {
      echo 'Build Failed'
    }
    always {
      step([$class: 'CoberturaPublisher', coberturaReportFile: 'target/site/cobertura/coverage.xml'])

      publishHTML target: [
        allowMissing: false,
        alwaysLinkToLastBuild: false,
        keepAll: false,
        reportDir: 'target/site/cobertura/',
        reportFiles: 'index.html',
        reportName: 'HTML Report'
      ]
    
      script {
          def mailRecipients = 'chandresh.kumar@guavus.com'
          def jobName = currentBuild.fullDisplayName
          emailext body: '''${JELLY_SCRIPT, template="user-management"}''',
          mimeType: 'text/html',
          attachmentsPattern: '*.png',
          subject: "[Jenkins] ${jobName}",
          to: "${mailRecipients}",
          replyTo: "${mailRecipients}"
      }
      slackSend channel: 'test-pipeline', message: "${currentBuild.currentResult}: ${env.JOB_NAME} ${env.BUILD_NUMBER}: ${env.BUILD_URL}"
    }
  }
}
