pipeline {
    agent { label 'test-frontend-agent' }

    triggers {
        GenericTrigger(
            genericVariables: [
                [key: 'RELEASE_TAG', value: '$.release.tag_name']
            ],
            causeString: 'Triggered by GitHub release: $RELEASE_TAG',
            token: 'github-release-trigger',
            printContributedVariables: true,
            printPostContent: true,
            regexpFilterText: '$.action',
            regexpFilterExpression: 'published'
        )
    }

    environment {
        BASE_PATH = "/var/www/admin/httpdocs"
        LIVE_DIR = "${BASE_PATH}/SFI-Admin-UI"
        REPO_OWNER = "Hari979"
        REPO_NAME = "crispy_kitchen"
    }

    stages {
        stage('Backup Current Deployment') {
            steps {
                script {
                    def TAG = env.RELEASE_TAG
                    def DATE = sh(script: "date +%F", returnStdout: true).trim()
                    def COUNT = 1
                    while (fileExists("${BASE_PATH}/SFI-Admin-UI-${DATE}-${COUNT}")) {
                        COUNT++
                    }
                    def BACKUP_DIR = "${BASE_PATH}/SFI-Admin-UI-${DATE}-${COUNT}"
                    echo "Backing up ${env.LIVE_DIR} to ${BACKUP_DIR}"
                    sh """
                        if [ -d "${env.LIVE_DIR}" ]; then
                            sudo mv "${env.LIVE_DIR}" "${BACKUP_DIR}"
                        fi
                        sudo mkdir -p "${env.LIVE_DIR}"
                    """
                }
            }
        }

        stage('Download and Deploy') {
            steps {
                script {
                    def TAG = env.RELEASE_TAG
                    def WORK_DIR = "/home/ubuntu/deploy-${TAG}"
                    def ZIP_PATH = "${WORK_DIR}/release.zip"
                    sh """
                        rm -rf "${WORK_DIR}"
                        mkdir -p "${WORK_DIR}"
                        curl -L -o "${ZIP_PATH}" \
                          "https://github.com/${env.REPO_OWNER}/${env.REPO_NAME}/archive/refs/tags/${TAG}.zip"
                        unzip "${ZIP_PATH}" -d "${WORK_DIR}"
                    """
                    def EXTRACTED_FOLDER = sh(
                        script: "find ${WORK_DIR} -maxdepth 1 -type d -name '${env.REPO_NAME}-*'",
                        returnStdout: true
                    ).trim()
                    sh """
                        sudo rsync -av "${EXTRACTED_FOLDER}/" "${env.LIVE_DIR}/"
                        rm -rf "${WORK_DIR}"
                        sudo systemctl restart apache2
                    """
                }
            }
        }
    }
}
