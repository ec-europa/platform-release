properties([
  parameters([
    string(name: 'GITHUB_PR_ID', defaultValue: '', description: 'Please enter the pull request ID from which to build the tag.'),
    string(name: 'GITHUB_RELEASE_TAG', defaultValue: '', description: 'Please enter the tag version for the release.'),
    string(name: 'GITHUB_RELEASE_TITLE', defaultValue: '', description: 'Please enter the release title.'),
    booleanParam(name: 'GITHUB_RELEASE_STATE', defaultValue: true, description: 'Please identify the release state. By default the release is set as a pre-release.'),
  ])
])

node('release') {
  env.GITHUB_RELEASE_FILE_NAME = env.GITHUB_REPO_NAME + "-" + env.GITHUB_RELEASE_TAG + ".tar.gz"
  env.GITHUB_RELEASE_FILE_PATH = env.RELEASE_PATH + "/" + env.GITHUB_RELEASE_FILE_NAME
  env.GITHUB_RELEASE_CHANGELOG_PATH = env.RELEASE_PATH + "/CHANGELOG-" + env.GITHUB_RELEASE_TAG + ".txt"
  env.GITHUB_PR_INFO_FILE_PATH = "/tmp/${GITHUB_REPO_NAME}-PR-${GITHUB_PR_ID}"
  env.slackMessage = "<${env.BUILD_URL}|Release ${env.GITHUB_RELEASE_TITLE} build>"
  slackSend color: "good", message: "${env.slackMessage} started."
  withCredentials([
    [$class: 'UsernamePasswordMultiBinding', credentialsId: 'GITHUB_REPO_AUTH', usernameVariable: 'GITHUB_REPO_USER', passwordVariable: 'GITHUB_REPO_TOKEN']
  ]) {
    try {
      stage ('Init') {
          env.GITHUB_API_BASE_URL = "https://api.github.com/repos/${GITHUB_REPO_USER}/${GITHUB_REPO_NAME}"
          sh "curl -s -o ${GITHUB_PR_INFO_FILE_PATH} ${GITHUB_API_BASE_URL}/pulls/${GITHUB_PR_ID}"
          env.GIT_REF = sh(
            script: "jq -r '.merge_commit_sha' ${GITHUB_PR_INFO_FILE_PATH}",
            returnStdout: true
          ).trim()
          env.GITHUB_RELEASE_DESC = sh(
            script: "jq -r '.body' ${GITHUB_PR_INFO_FILE_PATH}",
            returnStdout: true
          ).trim()
          env.GITHUB_BRANCH = sh(
            script:"jq -r '.base.ref' ${GITHUB_PR_INFO_FILE_PATH}",
            returnStdout: true
          ).trim()
      }
      stage ('Build package') {
          git([url: "${env.GITHUB_REPO_URL}", branch: "${env.GITHUB_BRANCH}", credentialsId: 'GITHUB_REPO_AUTH'])
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
          --label "${GITHUB_RELEASE_FILE_NAME}" \
          --file ${GITHUB_RELEASE_FILE_PATH}'''
      }
      stage ('Upload changelog') {
          sh '''github-release info \
            --security-token ${GITHUB_REPO_TOKEN} \
            --user ${GITHUB_REPO_USER} \
            --repo ${GITHUB_REPO_NAME} \
            --json | jq --arg x 'T' -r '.Releases[] | select(.id >= 5208235) | "# \\(.name), \\(.created_at | split($x)[0])\\n\\(.body)\\n"' > ${GITHUB_RELEASE_CHANGELOG_PATH}'''
          sh '''github-release upload \
            --security-token ${GITHUB_REPO_TOKEN} \
            --user ${GITHUB_REPO_USER} \
            --repo ${GITHUB_REPO_NAME} \
            --tag ${GITHUB_RELEASE_TAG} \
            --name "CHANGELOG.md" \
            --label "CHANGELOG.md" \
            --file ${GITHUB_RELEASE_CHANGELOG_PATH}'''
      }
    }
    catch (err) {
      slackSend color: "danger", message: "${env.slackMessage} failed."
      throw err
    }
  }
  slackSend color: "good", message: "${env.slackMessage} complete."
}
