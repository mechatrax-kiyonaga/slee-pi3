#!/usr/bin/env groovy
@Library('jenkins-custom-library')_
pipeline{
	agent any
	environment {
		JENKINS_CREDENTIALS = credentials('5b25baba-433c-41a9-b104-59e49ec74e49')
		FTP_CREDENTIALS = credentials('aa4ddfcc-11d9-418d-b794-8963612b6a78')
		FTP_IP_CREDENTIALS = credentials('0e3efab3-616e-439d-806b-55aac4cd84fd')

		TEMP_DIR = '/dev/shm/raspios'

		ZULIP_PROPS = readProperties file: "${JENKINS_HOME}/workspace/properties/zulip_prop"
		MAIN_STREAM = "${ZULIP_PROPS['MAIN_STREAM']}"
	}
	stages {
		stage("Checkout GIT") {
			steps {
				script {
					def vars = checkout(scm: [$class: 'GitSCM',
						branches: [[name: 'main']],
						userRemoteConfigs: [[url: 'https://github.com/mechatrax/slee-pi3']]
					], poll: true)
					def msg = sh(
						script: 'git log --decorate=no --oneline | sed -e \'s/[0-9a-f]\\+ //\' | grep ^Release | head -1',
						returnStdout: true
					).trim()
					if ( ! msg.contains("Release") ) {
						sh 'curl -s -X POST -u ${JENKINS_CREDENTIALS_USR}:${JENKINS_CREDENTIALS_PSW} http://127.0.0.1:8080/job/${JOB_NAME}/${BUILD_NUMBER}/stop'
					}
					env.RELEASE_NAME = msg.replaceFirst('Release.*(sleepi3[^ ]+).*', '$1')
					if ( env.RELEASE_NAME == "" ) {
						sh 'curl -s -X POST -u ${JENKINS_CREDENTIALS_USR}:${JENKINS_CREDENTIALS_PSW} http://127.0.0.1:8080/job/${JOB_NAME}/${BUILD_NUMBER}/stop'
					}
					if ( msg.contains('legacy') ) {
						env.RELEASE_SUFFIX = '-legacy'
						env.RELEASE_MODIFIER = ' Legacy 版'
						env.SUM_FILE = 'sleepi3_oldstable_lite_armhf.sha256sum'
					} else {
						env.RELEASE_SUFFIX = ''
						env.RELEASE_MODIFIER = ''
						env.SUM_FILE = 'sleepi3_lite_arm64.sha256sum'
					}
					env.TWEET_MESSAGE = "弊社ラズベリーパイ用電源監視/死活監視モジュール slee-Pi（スリーピー）の${RELEASE_MODIFIER} OS イメージ ${RELEASE_NAME} をリリースしました。\\nhttps://github.com/mechatrax/slee-pi3/blob/main/os${RELEASE_SUFFIX}/${RELEASE_NAME}.md"
				}
			}
		}
		stage("Validate") {
			when {
				expression { return RELEASE_NAME != 'none' }
			}
			steps {
				script {
					def latest_release = sh(script: 'curl -sS -u ${FTP_CREDENTIALS_USR}:${FTP_CREDENTIALS_PSW} ftp://${FTP_IP_CREDENTIALS}/data/slee-pi3${RELEASE_SUFFIX}/ | grep -o \'sleepi3[^\"]*-[0-9]\\{8\\}\\.img\\.xz\' | sort -t \'-\' -k5n | tail -n1 | awk -F. \'{print \$1}\'', returnStdout: true).trim()
					if (latest_release == RELEASE_NAME) {
						sh 'curl -s -X POST -u ${JENKINS_CREDENTIALS_USR}:${JENKINS_CREDENTIALS_PSW} http://127.0.0.1:8080/job/${JOB_NAME}/${BUILD_NUMBER}/stop'
					}
				}
			}
		}
		stage("Upload") {
			when {
				expression { return RELEASE_NAME != 'none' }
			}
			steps {
				sh 'curl -sS -T ${TEMP_DIR}/${RELEASE_NAME}.img.xz -u ${FTP_CREDENTIALS_USR}:${FTP_CREDENTIALS_PSW} ftp://${FTP_IP_CREDENTIALS}/data/slee-pi3${RELEASE_SUFFIX}/'
			}
		}
		stage("Check") {
			when {
				expression { return RELEASE_NAME != 'none' }
			}
			steps {
				sh 'test -e ${TEMP_DIR}/${RELEASE_NAME}.img.xz && sudo mv ${TEMP_DIR}/${RELEASE_NAME}.img.xz ${TEMP_DIR}/${RELEASE_NAME}.img.xz.orig'
				sh 'wget -q https://mechatrax.com/data/slee-pi3${RELEASE_SUFFIX}/${RELEASE_NAME}.img.xz -P ${TEMP_DIR}'
				sh 'sha256sum -c ${TEMP_DIR}/${SUM_FILE}'
				sh 'sudo rm -vf ${TEMP_DIR}/${RELEASE_NAME}.img.xz ${TEMP_DIR}/${RELEASE_NAME}.img.xz.orig ${TEMP_DIR}/${SUM_FILE}'
				sh 'test -e ${TEMP_DIR} && sudo rm -rf ${TEMP_DIR}/*'
			}
		}
		stage("Tweet") {
			when {
				expression { return RELEASE_NAME != 'none' }
			}
			steps {
				withCredentials([
					usernamePassword(credentialsId: 'c4c404b1-66c8-4dac-8c52-fdf8c4b8b700', passwordVariable: 'NAME', usernameVariable: 'BEARER'),
					usernamePassword(credentialsId: '5862efc0-4e22-4447-9ac4-af71c72f7fea', passwordVariable: 'APISECRET', usernameVariable: 'API'),
					usernamePassword(credentialsId: '555f6776-fbe8-4833-b8dd-cf9e4838a7d6', passwordVariable: 'ACCESSSECRET', usernameVariable: 'ACCESS')
				]) {
					sh 'python3 -c "import tweepy; tweepy.Client(bearer_token=\\"${BEARER}\\", consumer_key=\\"${API}\\",consumer_secret=\\"${APISECRET}\\",access_token=\\"${ACCESS}\\",access_token_secret=\\"${ACCESSSECRET}\\").create_tweet(text=\\"${TWEET_MESSAGE}\\")"'
				}
			}
		}
	}
	post {
		success {
			zulipSend message: "✔ slee-Pi3 の SD イメージ ${RELEASE_NAME} のリリースに成功しました", stream: "${MAIN_STREAM}"
		}
		failure {
			zulipSend message: "❌ slee-Pi3 の SD イメージ ${RELEASE_NAME} のリリースに失敗しました", stream: "${MAIN_STREAM}"
		}
	}
}
