// Licensed to the Apache Software Foundation (ASF) under one
// or more contributor license agreements.  See the NOTICE file
// distributed with this work for additional information
// regarding copyright ownership.  The ASF licenses this file
// to you under the Apache License, Version 2.0 (the
// "License"); you may not use this file except in compliance
// with the License.  You may obtain a copy of the License at
//
//   http://www.apache.org/licenses/LICENSE-2.0
//
// Unless required by applicable law or agreed to in writing,
// software distributed under the License is distributed on an
// "AS IS" BASIS, WITHOUT WARRANTIES OR CONDITIONS OF ANY
// KIND, either express or implied.  See the License for the
// specific language governing permissions and limitations
// under the License.
pipeline {

  agent {
    // Hadoop and ubuntu for ASF, rest are private
    label 'Hadoop||ubuntu||azaka||small'
  }

  options {
    buildDiscarder(logRotator(numToKeepStr: '5'))
    timeout (time: 9, unit: 'HOURS')
    timestamps()
    checkoutToSubdirectory('src')
  }

  environment {
    YETUS_BASEDIR = "${WORKSPACE}/yetus"
    // will also need to change notification section below
    YETUS_RELATIVE_PATCHDIR = 'out'
    YETUS_DOCKERFILE = "${YETUS_BASEDIR}/precommit/src/main/shell/test-patch-docker/Dockerfile"
    SRCDIR="${WORKSPACE}/src"
  }

  parameters {
    string(name: 'ISSUE_NUM',
           defaultValue: '',
           description: 'The JIRA YETUS issue number that has a patch needing pre-commit testing. Example: 1234')
    booleanParam(name: 'USE_DEBUG_FLAG',
                 defaultValue: false,
                 description: 'click to enable extra outputs')
    booleanParam(name: 'USE_DOCKER_FLAG',
                 defaultValue: true,
                 description: 'Disable to turn off Docker mode')
    string(name: 'EXTRA_ARGS',
           defaultValue: '',
           description: 'Any extra test-patch arguments')
  }

  stages {

    stage ('extract yetus') {
      steps {
        dir("${YETUS_BASEDIR}") {
          checkout([
            $class: 'GitSCM',
            branches: [[ name: 'yetus1079' ]],
            userRemoteConfigs: [[
              url: 'https://github.com/aw-was-here/buretoolbox',
              credentialsId: 'apache-yetus-at-github.com'
            ]]
          ])
        }
      }
    }
    stage ('precommit-run') {
      steps {
        withCredentials([usernamePassword(credentialsId: 'apache-yetus-at-github.com',
                         passwordVariable: 'GITHUB_TOKEN',
                         usernameVariable: 'GITHUB_USER')]) {
          sh '''#!/usr/bin/env bash

              USE_DOCKER_FLAG=${USE_DOCKER_FLAG:-true}

              # clean and make a new directory for our output artifacts, temporary
              # storage, etc just in case the workspace directory
              # delete in post is removed
              if [[ -d "${WORKSPACE}/${YETUS_RELATIVE_PATCHDIR}" ]]; then
                rm -rf "${WORKSPACE}/${YETUS_RELATIVE_PATCHDIR}"
              fi
              mkdir -p "${WORKSPACE}/${YETUS_RELATIVE_PATCHDIR}"
              YETUS_ARGS+=("--patch-dir=${WORKSPACE}/${YETUS_RELATIVE_PATCHDIR}")

              # where the source is located
              YETUS_ARGS+=("--basedir=${SRCDIR}")

              # lots of different output formats
              YETUS_ARGS+=("--brief-report-file=${WORKSPACE}/${YETUS_RELATIVE_PATCHDIR}/brief.txt")
              YETUS_ARGS+=("--console-report-file=${WORKSPACE}/${YETUS_RELATIVE_PATCHDIR}/console.txt")
              YETUS_ARGS+=("--html-report-file=${WORKSPACE}/${YETUS_RELATIVE_PATCHDIR}/report.html")
              YETUS_ARGS+=("--junit-report-xml=${WORKSPACE}/${YETUS_RELATIVE_PATCHDIR}/junit-report.xml")

              # enable writing back to Github
              YETUS_ARGS+=(--github-token="${GITHUB_TOKEN}")


              # auto-kill any surefire stragglers during unit test runs
              YETUS_ARGS+=(--reapermode=report)

              # URL for user-side presentation in reports and such to our artifacts
              # (needs to match the archive bits below)
              YETUS_ARGS+=("--build-url-artifacts=artifact/out")

              # plugins to enable
              YETUS_ARGS+=("--plugins=all")

              if [[ "${USE_DEBUG_FLAG}" == true ]]; then
                YETUS_ARGS+=("--debug")
              fi

              # run in docker mode and specifically point to our
              # Dockerfile since we don't want to use the auto-pulled version.
              if [[ "${USE_DOCKER_FLAG}" == true ]]; then
                docker pull ubuntu:focal
                YETUS_ARGS+=("--docker")
                YETUS_ARGS+=("--dockerfile=${YETUS_DOCKERFILE}")
                YETUS_ARGS+=("--docker-cache-from=apache/yetus-base:main")
              fi

              # run test-patch from the source tree specified up above
              TESTPATCHBIN=${YETUS_BASEDIR}/precommit/src/main/shell/test-patch.sh

              # execute! (we are using bash instead of the
              # bin in case the perms get messed up)
              /usr/bin/env bash "${TESTPATCHBIN}" "${YETUS_ARGS[@]}" ${EXTRA_ARGS}

              '''
        }
      }
    }
  }
  post {
    always {
      script {

        // Publish status if it was missed
        withCredentials([usernamePassword(credentialsId: 'apache-yetus-at-github.com',
                         passwordVariable: 'GITHUB_TOKEN',
                         usernameVariable: 'GITHUB_USER')]) {
            sh '''#!/usr/bin/env bash

                # enable writing back to Github
                YETUS_ARGS+=(--github-token="${GITHUB_TOKEN}")
                YETUS_ARGS+=("--patch-dir=${WORKSPACE}/${YETUS_RELATIVE_PATCHDIR}")

                if [[ "${USE_DEBUG_FLAG}" == true ]]; then
                  YETUS_ARGS+=("--debug")
                fi

                # run test-patch from the source tree specified up above
                TESTPATCHBIN=${YETUS_BASEDIR}/precommit/src/main/shell/github-status-recovery.sh

                # execute! (we are using bash instead of the
                # bin in case the perms get messed up)
                /usr/bin/env bash "${TESTPATCHBIN}" "${YETUS_ARGS[@]}" ${EXTRA_ARGS} || true

                '''
        }

        // Publish JUnit results
        try {
            junit "${env.YETUS_RELATIVE_PATCHDIR}/junit-report.xml"
        } catch(e) {
            echo 'junit processing: ' + e.toString()
        }
        archiveArtifacts "${env.YETUS_RELATIVE_PATCHDIR}/**"
        // Publish the HTML report so that it can be looked at
        // Has to be relative to WORKSPACE.
        publishHTML (target: [
                      allowMissing: true,
                      keepAll: true,
                      alwaysLinkToLastBuild: true,
                      // Has to be relative to WORKSPACE
                      reportDir: "${env.YETUS_RELATIVE_PATCHDIR}",
                      reportFiles: 'report.html',
                      reportName: 'Yetus Report'
        ])
        if ((env.BRANCH_NAME == 'main') && (env.BUILD_URL.contains('apache.org'))) {
          emailext(subject: '$DEFAULT_SUBJECT',
                   body:
'''For more details, see ${BUILD_URL}

${CHANGES, format="[%d] (%a) %m"}

HTML Version: ${BUILD_URL}Yetus_20Report/


${FILE,path="out/brief.txt"}

''',
                  replyTo: 'dev@yetus.apache.org',
                  to: 'dev@yetus.apache.org'
          )
        }
      }
    }

    // on failure, we send an email to the person who changed
    // the code and the person who requested the job to get run
    failure {
      script {
        if (env.BUILD_URL.contains('apache.org')) {
          emailext(subject: '$DEFAULT_SUBJECT',
                  body:
'''For more details, see ${BUILD_URL}

${CHANGES, format="[%d] (%a) %m"}

HTML Version: ${BUILD_URL}Yetus_20Report/

${FILE,path="out/brief.txt"}
''',
                  recipientProviders: [
                    [$class: 'DevelopersRecipientProvider'],
                    [$class: 'RequesterRecipientProvider']
                  ],
                  replyTo: '$DEFAULT_REPLYTO',
                  to: '$DEFAULT_RECIPIENTS'
          )
        }
      }
    }

    // Jenkins has issues in some configurations sending
    // signals to child processes. Additionally, Github Branch
    // Source plug-in will quickly fill Jenkins build hosts
    // with PR directories.  This cleanup stanze kills
    // any left over processes/containers and frees disk space
    // on exit
    cleanup() {
      script {
        sh '''
            if [ -f "${WORKSPACE}/${PATCHDIR}/pidfile.txt" ]; then
              echo "test-patch process appears to still be running: killing"
              kill `cat "${WORKSPACE}/${PATCHDIR}/pidfile.txt"` || true
              sleep 10
            fi
            if [ -f "${WORKSPACE}/${PATCHDIR}/cidfile.txt" ]; then
              echo "test-patch container appears to still be running: killing"
              docker kill `cat "${WORKSPACE}/${PATCHDIR}/cidfile.txt"` || true
            fi
            '''
        deleteDir()
      }
    }
  }
}
