#!groovy

pipeline {
	agent {
		dockerfile {
			filename 'Dockerfile.jenkins'
			// 'docker build' would normally copy the whole build-dir to the container, changing the
			// docker build directory avoids that overhead
			dir 'docker'
			// Pass the uid and the gid of the current user (jenkins-user) to the Dockerfile, so a
			// corresponding user can be added. This is needed to provide the jenkins user inside
			// the container for the ssh-agent to work.
			// Another way would be to simply map the passwd file, but would spoil additional information
			additionalBuildArgs '--build-arg USER_ID=$(id -u) --build-arg GROUP_ID=$(id -g)'
			// Currently there are two different NDK behaviors in place, one to keep NDK r16b, which
			// was needed because of the removal of armeabi and MIPS support and one to always use the
			// latest NDK, which is the suggestion from the NDK documentations.
			// Therefore two different SDK locations on the host are currently in place:
			// NDK r16b  : /var/local/container_shared/android-sdk
			// NDK latest: /var/local/container_shared/android-sdk-ndk-latest
			// As android-sdk was used from the beginning and is already 'released' this can't be changed
			// to eg android-sdk-ndk-r16b and must be kept to the previously used value
			args "--device /dev/kvm:/dev/kvm -v /var/local/container_shared/gradle/:/.gradle -v /var/local/container_shared/android-sdk-ndk-latest:/usr/local/android-sdk -v /var/local/container_shared/android-home:/.android -v /var/local/container_shared/emulator_console_auth_token:/.emulator_console_auth_token -v /var/local/container_shared/analytics.settings:/analytics.settings"
		}
	}

	environment {
		//////// Define environment variables to point to the correct locations inside the container ////////
		//////////// Most likely not edited by the developer
		ANDROID_SDK_ROOT = "/usr/local/android-sdk"
		// Deprecated: Still used by the used gradle version, once gradle respects ANDROID_SDK_ROOT, this can be removed
		ANDROID_HOME = "/usr/local/android-sdk"
		ANDROID_SDK_HOME = "/"
		// Needed for compatibiliby to current Jenkins-wide Envs. Can be removed, once all builds are migrated to Pipeline
		ANDROID_SDK_LOCATION = "/usr/local/android-sdk"
		ANDROID_NDK = ""
		// This is important, as we want the keep our gradle cache, but we can't share it between containers
		// the cache could only be shared if the gradle instances could comunicate with each other
		// imho keeping the cache per executor will have the least space impact
		GRADLE_USER_HOME = "/.gradle/${env.EXECUTOR_NUMBER}"
		// Otherwise user.home returns ? for java applications
		JAVA_TOOL_OPTIONS = "-Duser.home=/tmp/"

		//////// Build specific variables ////////
		//////////// May be edited by the developer on changing the build steps
		// modulename
		GRADLE_PROJECT_MODULE_NAME = "catroid"

		// APK build output locations
		APK_LOCATION_DEBUG = "${env.GRADLE_PROJECT_MODULE_NAME}/build/outputs/apk/catroid/debug/catroid-catroid-debug.apk"
		APK_LOCATION_STANDALONE = "${env.GRADLE_PROJECT_MODULE_NAME}/build/outputs/apk/standalone/debug/catroid-standalone-debug.apk"

		JACOCO_XML = "${env.GRADLE_PROJECT_MODULE_NAME}/build/reports/coverage/catroid/debug/report.xml"
		JACOCO_UNIT_XML = "${env.GRADLE_PROJECT_MODULE_NAME}/build/reports/jacoco/jacocoTestCatroidDebugUnitTestReport/jacocoTestCatroidDebugUnitTestReport.xml"

		// place the cobertura xml relative to the source, so that the source can be found
		JAVA_SRC = "${env.GRADLE_PROJECT_MODULE_NAME}/src/main/java"
	}

	options {
		timeout(time: 2, unit: 'HOURS')
		timestamps()
		buildDiscarder(logRotator(numToKeepStr: '30'))
	}

    triggers {
        cron(env.BRANCH_NAME == 'develop' ? '@midnight' : '')
    }

	stages {
		stage('Setup Android SDK') {
			steps {
				// Install Android SDK
				lock("update-android-sdk-on-${env.NODE_NAME}") {
					sh './gradlew -PinstallSdk'
				}
			}
		}

		stage('Static Analysis') {
			steps {
				sh './gradlew pmd checkstyle lint'
			}

			post {
				always {
					pmd         canComputeNew: false, canRunOnFailed: true, defaultEncoding: '', healthy: '', pattern: "${env.GRADLE_PROJECT_MODULE_NAME}/build/reports/pmd.xml",        unHealthy: '', unstableTotalAll: '0'
					checkstyle  canComputeNew: false, canRunOnFailed: true, defaultEncoding: '', healthy: '', pattern: "${env.GRADLE_PROJECT_MODULE_NAME}/build/reports/checkstyle.xml", unHealthy: '', unstableTotalAll: '0'
					androidLint canComputeNew: false, canRunOnFailed: true, defaultEncoding: '', healthy: '', pattern: "${env.GRADLE_PROJECT_MODULE_NAME}/build/reports/lint*.xml",      unHealthy: '', unstableTotalAll: '0'
				}
			}
		}

		stage('Unit and Device tests') {
			steps {
				// Run local unit tests
				sh './gradlew -PenableCoverage jacocoTestCatroidDebugUnitTestReport'
				// Convert the JaCoCo coverate to the Cobertura XML file format.
				// This is done since the Jenkins JaCoCo plugin does not work well.
				// See also JENKINS-212 on jira.catrob.at
				sh "if [ -f '$JACOCO_UNIT_XML' ]; then ./buildScripts/cover2cover.py $JACOCO_UNIT_XML > $JAVA_SRC/coverage1.xml; fi"

				// Run device tests for package: org.catrobat.catroid.test
				sh '''./gradlew -PenableCoverage -Pemulator=android24 startEmulator createCatroidDebugAndroidTestCoverageReport \
							-Pandroid.testInstrumentationRunnerArguments.package=org.catrobat.catroid.test'''
				// Convert the JaCoCo coverate to the Cobertura XML file format.
				// This is done since the Jenkins JaCoCo plugin does not work well.
				// See also JENKINS-212 on jira.catrob.at
				sh "if [ -f '$JACOCO_XML' ]; then ./buildScripts/cover2cover.py $JACOCO_XML > $JAVA_SRC/coverage2.xml; fi"
				// ensure that the following test run does not overwrite the results
				sh "mv ${env.GRADLE_PROJECT_MODULE_NAME}/build/outputs/androidTest-results ${env.GRADLE_PROJECT_MODULE_NAME}/build/outputs/androidTest-results1"

				// Run device tests for class: org.catrobat.catroid.uiespresso.testsuites.PullRequestTriggerSuite
				sh '''./gradlew -PenableCoverage -Pemulator=android24 startEmulator createCatroidDebugAndroidTestCoverageReport \
							-Pandroid.testInstrumentationRunnerArguments.class=org.catrobat.catroid.uiespresso.testsuites.PullRequestTriggerSuite'''
				// Convert the JaCoCo coverate to the Cobertura XML file format.
				// This is done since the Jenkins JaCoCo plugin does not work well.
				// See also JENKINS-212 on jira.catrob.at
				sh "if [ -f '$JACOCO_XML' ]; then ./buildScripts/cover2cover.py $JACOCO_XML > $JAVA_SRC/coverage3.xml; fi"
				// ensure that the following test run does not overwrite the results
				sh "mv ${env.GRADLE_PROJECT_MODULE_NAME}/build/outputs/androidTest-results ${env.GRADLE_PROJECT_MODULE_NAME}/build/outputs/androidTest-results2"
			}

			post {
				always {
					sh './gradlew stopEmulator clearAvdStore'
					archiveArtifacts 'logcat.txt'
				}
			}
		}

		stage('Quarantined Tests') {
			when {
				expression { isJobStartedByTimer() }
			}

			steps {
				sh '''./gradlew -PenableCoverage -PlogcatFile=quarantined_logcat.txt -Pemulator=android24 \
							startEmulator createCatroidDebugAndroidTestCoverageReport \
							-Pandroid.testInstrumentationRunnerArguments.class=org.catrobat.catroid.uiespresso.testsuites.QuarantineTestSuite'''
				archiveArtifacts "$JACOCO_XML"
				sh "if [ -f '$JACOCO_XML' ]; then ./buildScripts/cover2cover.py $JACOCO_XML > $JAVA_SRC/coverage4.xml; fi"

			}

			post {
				always {
					sh './gradlew stopEmulator clearAvdStore'
					archiveArtifacts 'quarantined_logcat.txt'
				}
			}
		}

		stage('Standalone-APK') {
			// It checks that the creation of standalone APKs (APK for a Pocketcode app) works, reducing the risk of breaking gradle changes.
			// The resulting APK is not verified itself.
			steps {
				sh '''./gradlew assembleStandaloneDebug -Papk_generator_enabled=true -Psuffix=generated817.catrobat \
							-Pdownload='https://share.catrob.at/pocketcode/download/817.catrobat' '''
				archiveArtifacts "${env.APK_LOCATION_STANDALONE}"
			}
		}

		stage('Independent-APK') {
			// It checks that the job builds with the parameters to have unique APKs, reducing the risk of breaking gradle changes.
			// The resulting APK is not verified on itself.
			steps {
				sh "./gradlew assembleCatroidDebug -Pindependent='Code Nightly #$BUILD_NUMBER'"
				archiveArtifacts "${env.APK_LOCATION_DEBUG}"
			}
		}
	}

	post {
		always {
			junit '**/*TEST*.xml'
			cobertura autoUpdateHealth: false, autoUpdateStability: false, coberturaReportFile: "$JAVA_SRC/coverage*.xml", failUnhealthy: false, failUnstable: false, maxNumberOfBuilds: 0, onlyStable: false, sourceEncoding: 'ASCII', zoomCoverageChart: false, failNoReports: false
			step([$class: 'LogParserPublisher', failBuildOnError: true, projectRulePath: 'buildScripts/log_parser_rules', unstableOnWarning: true, useProjectRule: true])

			// Send notifications with standalone=false
			script {
				sendNotifications false
			}
		}
	}
}
