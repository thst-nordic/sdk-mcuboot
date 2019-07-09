
@Library("CI_LIB@lib_master") _

def IMAGE_TAG = lib_Main.getDockerImage(JOB_NAME)
def AGENT_LABELS = lib_Main.getAgentLabels(JOB_NAME)
def REPO_CI_TOOLS = "https://github.com/zephyrproject-rtos/ci-tools.git"
def REPO_CI_TOOLS_SHA = "bfe635f102271a4ad71c1a14824f9d5e64734e57"
def CI_STATE = new HashMap()

pipeline {
  agent {
    docker {
      image IMAGE_TAG
      label AGENT_LABELS
    }
  }
  options {
    // Checkout the repository to this folder instead of root
    checkoutToSubdirectory('mcuboot')
  }

  environment {
      // This token is used to by check_compliance to comment on PRs and use checks
      GH_TOKEN = credentials('nordicbuilder-compliance-token')
      GH_USERNAME = "NordicBuilder"
      COMPLIANCE_SCRIPT_PATH = "../ci-tools/scripts/check_compliance.py"
      COMPLIANCE_ARGS = "-r NordicPlayground/fw-nrfconnect-mcuboot"
      COMPLIANCE_REPORT_ARGS = "-p $CHANGE_ID -S $GIT_COMMIT -g"
  }

  stages {
    stage('Load') {
      steps {
        println "Using Node:$NODE_NAME and Input Parameters:$params"
        println "Input CI_STATE:${params['jsonstr_CI_STATE']}"
        dir("ci-tools") {
          git branch: "master", url: "$REPO_CI_TOOLS"
          sh "git checkout ${REPO_CI_TOOLS_SHA}"
        }

        script {
          CI_STATE = lib_State.load(params['jsonstr_CI_STATE'])
          lib_State.store('MCUBOOT', CI_STATE)
          lib_State.getParentJob(CI_STATE)
          lib_State.pushjobStack('MCUBOOT', CI_STATE)
          println "CI_STATE = $CI_STATE"
        }
      }
    }
    stage('Checkout repositories') {
      when { expression {  true || CI_STATE.MCUBOOT.RUN_TESTS || CI_STATE.MCUBOOT.RUN_BUILD } }
      steps {
        script {
          CI_STATE.MCUBOOT.REPORT_SHA = lib_Main.checkoutRepo(CI_STATE.MCUBOOT.GIT_URL, "zephyr", CI_STATE, false)
          println "CI_STATE.MCUBOOT.REPORT_SHA = " + CI_STATE.MCUBOOT.REPORT_SHA
          lib_West.InitUpdate('zephyr')
        }
      }
    }
    stage('Apply Parent Manifest Updates') {
      when { expression { true || CI_STATE.MCUBOOT.RUN_TESTS || CI_STATE.MCUBOOT.RUN_BUILD } }
      steps {
        script {
          println "If triggered by an upstream Project, use their changes."
          lib_West.ApplyManfestUpdates(CI_STATE)
        }
      }
    }

    stage('Run compliance check') {
      when { expression { CI_STATE.MCUBOOT.RUN_TESTS } }
      steps {
        dir('mcuboot') {
          script {
            // If we're a pull request, compare the target branch against the current HEAD (the PR)
            if (env.CHANGE_TARGET) {
              COMMIT_RANGE = "origin/${env.CHANGE_TARGET}..HEAD"
              COMPLIANCE_ARGS = "$COMPLIANCE_ARGS $COMPLIANCE_REPORT_ARGS"
            }
            // If not a PR, it's a non-PR-branch or master build. Compare against the origin.
            else {
              COMMIT_RANGE = "origin/${env.BRANCH_NAME}..HEAD"
            }
            // Run the compliance check
            try {
              sh "$COMPLIANCE_SCRIPT_PATH $COMPLIANCE_ARGS --commits $COMMIT_RANGE"
            }
            finally {
              junit 'compliance.xml'
              archiveArtifacts artifacts: 'compliance.xml'
            }
          }
        }
      }
    }
    stage('Build samples') {
      when { expression { CI_STATE.MCUBOOT.RUN_BUILD } }
      steps {
          echo "No Samples to build yet."
      }
    }
    stage('Trigger testing build') {
      when { expression { CI_STATE.MCUBOOT.RUN_DOWNSTREAM } }
      steps {
        script {
          CI_STATE.MCUBOOT.WAITING = true
          def DOWNSTREAM_JOBS = lib_Main.getDownStreamJobs(CI_STATE, 'MCUBOOT')
          println "DOWNSTREAM_JOBS = " + DOWNSTREAM_JOBS
          lib_West.AddManifestUpdate("MCUBOOT", 'mcuboot', CI_STATE.MCUBOOT.GIT_URL, CI_STATE.MCUBOOT.REPORT_SHA, CI_STATE)
          def jobs = [:]
          DOWNSTREAM_JOBS.each {
            jobs["${it}"] = {
              build job: "${it}", propagate: true, wait: true, parameters: [
                        string(name: 'jsonstr_CI_STATE', value: lib_Util.HashMap2Str(CI_STATE))]
            }
          }
          parallel jobs
        }
      }
    }

  }

  post {
    // This is the order that the methods are run. {always->success/abort/failure/unstable->cleanup}
    always {
      echo "always"
    }
    success {
      echo "success"
    }
    aborted {
      echo "aborted"
    }
    unstable {
      echo "unstable"
    }
    failure {
      echo "failure"
      script{
        if (env.BRANCH_NAME == 'master' || env.BRANCH_NAME.startsWith("PR"))
        {
            emailext(to: 'anpu',
                body: "${currentBuild.currentResult}\nJob ${env.JOB_NAME}\t\t build ${env.BUILD_NUMBER}\r\nLink: ${env.BUILD_URL}",
                subject: "[Jenkins][Build ${currentBuild.currentResult}: ${env.JOB_NAME}]",
                mimeType: 'text/html',)
        }
        else
        {
            echo "Branch ${env.BRANCH_NAME} is not master nor PR. Sending failure email skipped."
        }
      }
    }
    cleanup {
        echo "cleanup"
        // Clean up the working space at the end (including tracked files)
        cleanWs()
    }
  }
}
