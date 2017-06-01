properties([
  parameters([
    string(name: 'GIT_REF', defaultValue: '', description: 'Please enter the commit reference from which to build the tag.'),
    string(name: 'GITHUB_RELEASE_TAG', defaultValue: '', description: 'Please enter the tag version for the release.'),
    string(name: 'GITHUB_RELEASE_TITLE', defaultValue: '', description: 'Please enter the release title.'),
    text(name: 'GITHUB_RELEASE_DESC', defaultValue: '', description: 'Please describe this release. You can use the Markdown markup.'),
    booleanParam(name: 'GITHUB_RELEASE_STATE', defaultValue: true, description: 'Please identify the release state. By default the release is set as a pre-release.'),
  ])
])

node('release') {
  env.GITHUB_RELEASE_FILE_PATH = env.RELEASE_PATH + "/" + env.GITHUB_REPO_NAME + "-" + env.GITHUB_RELEASE_TAG + ".tar.gz"
  env.slackMessage = "<${env.BUILD_URL}|Release ${env.GITHUB_RELEASE_TITLE} build>"
  slackSend color: "good", message: "${env.slackMessage} started."
  withCredentials([
    [$class: 'UsernamePasswordMultiBinding', credentialsId: 'GITHUB_REPO_AUTH', usernameVariable: 'GITHUB_REPO_USER', passwordVariable: 'GITHUB_REPO_TOKEN']
  ]) {
    try {
      stage ('Build package') {
          git env.GITHUB_REPO_URL
          sh "git checkout ${GIT_REF}"
          sh "COMPOSER_CACHE_DIR=/dev/null composer install --no-suggest"
          sh "./bin/phing build-multisite-dist -Dcomposer.bin=`which composer`"
          sh "cd build && tar -czf ${GITHUB_RELEASE_FILE_PATH} ."
      }
      stage ('Create release') {
        def release_cmd = "github-release release" \
          + " --security-token ${GITHUB_REPO_TOKEN}" \
          + " --user ${GITHUB_REPO_USER}" \
          + " --repo ${GITHUB_REPO_NAME}" \
          + " --tag ${GITHUB_RELEASE_TAG}" \
          + " --name \"${GITHUB_RELEASE_TITLE}\"" \
          + " --description \"${GITHUB_RELEASE_DESC}\"" \
          + " --target ${GIT_REF}"

        def release_cmd_parameter = " --pre-release"
        def github_release_cmd = release_cmd

        if (params.GITHUB_RELEASE_STATE) {
          github_release_cmd = release_cmd + release_cmd_parameter
        }

        echo "Executing following command: \n${github_release_cmd}"
        sh github_release_cmd
      }
      
      stage ('Upload file') {
        sh '''github-release upload \
          --security-token ${GITHUB_REPO_TOKEN} \
          --user ${GITHUB_REPO_USER} \
          --repo ${GITHUB_REPO_NAME} \
          --tag ${GITHUB_RELEASE_TAG} \
          --name "${GITHUB_RELEASE_FILE_NAME}" \
          --label "${GITHUB_RELEASE_FILE_DESC}" \
          --file ${GITHUB_RELEASE_FILE_PATH}'''
      }
    }
    catch (err) {
      slackSend color: "danger", message: "${env.slackMessage} failed."
      throw err
    }
  }
  slackSend color: "good", message: "${env.slackMessage} complete."
}
