pipeline {
    agent none

    options {
        skipDefaultCheckout(true)
    }

    parameters {
        choice(
            name: 'PLATFORM',
            choices: ['ANDROID', 'IOS', 'BOTH'],
            description: 'Choose platform to build'
        )

        choice(
            name: 'BUILD_TYPE',
            choices: ['DEBUG', 'RELEASE'],
            description: 'Choose debug or release build'
        )

        choice(
            name: 'DISTRIBUTION',
            choices: ['NONE', 'FIREBASE', 'STORE'],
            description: 'Choose distribution target'
        )

        choice(
            name: 'ANDROID_ARTIFACT',
            choices: ['APK', 'AAB'],
            description: 'Android artifact type'
        )
        string(
                name: 'VERSION_NAME',
                defaultValue: '1.0.0',
                description: 'App version name'
        )

        string(
            name: 'VERSION_CODE',
            defaultValue: '',
            description: 'Android version code. Empty = Jenkins BUILD_NUMBER'
        )

        text(
            name: 'RELEASE_NOTES',
            defaultValue: 'SmartServe Flutter build',
            description: 'Release notes for Firebase'
        )
    }

    environment {
        FLUTTER_HOME = '/Users/enz/development/flutter'
        ANDROID_HOME = '/Users/enz/Library/Android/sdk'
        ANDROID_SDK_ROOT = '/Users/enz/Library/Android/sdk'
        PATH = "${FLUTTER_HOME}/bin:/opt/homebrew/bin:/usr/local/bin:${ANDROID_HOME}/cmdline-tools/latest/bin:${ANDROID_HOME}/platform-tools:/usr/bin:/bin:/usr/sbin:/sbin:${env.PATH}"    }

    stages {
        stage('Validate Parameters') {
            agent { label 'mac' }
            steps {
                script {
                    if (params.BUILD_TYPE == 'DEBUG' && params.DISTRIBUTION != 'NONE') {
                        error("DEBUG build should use DISTRIBUTION=NONE")
                    }

                    if (params.DISTRIBUTION == 'STORE' && params.BUILD_TYPE != 'RELEASE') {
                        error("STORE distribution requires RELEASE build")
                    }

                    if (params.DISTRIBUTION == 'FIREBASE' && params.ANDROID_ARTIFACT != 'APK') {
                        error("Firebase distribution should use ANDROID_ARTIFACT=APK unless Firebase is linked to Google Play.")
                    }
                }
            }
        }

        stage('Prepare Version and Release Notes') {
            agent { label 'mac' }

            steps {
                script {
                    env.APP_VERSION_NAME = params.VERSION_NAME?.trim() ?: "1.0.${env.BUILD_NUMBER}"
                    env.APP_VERSION_CODE = params.VERSION_CODE?.trim() ?: env.BUILD_NUMBER

                    def notes = params.RELEASE_NOTES?.trim()
                    if (!notes) {
                        notes = "SmartServe build #${env.BUILD_NUMBER}"
                    }

                    writeFile file: 'release-notes.txt', text: notes + "\n"

                    echo "Version name: ${env.APP_VERSION_NAME}"
                    echo "Version code: ${env.APP_VERSION_CODE}"
                    echo "Using release notes:"
                    echo readFile('release-notes.txt')
                }

                stash name: 'release-notes', includes: 'release-notes.txt'
            }
        }

        stage('Parallel Build') {
            steps {
                script {
                    def branches = [:]

                    if (params.PLATFORM == 'ANDROID' || params.PLATFORM == 'BOTH') {
                        branches['Android Build'] = {
                            node('mac') {
                                ws("${env.WORKSPACE}@android") {
                                    checkout scm
                                    unstash 'release-notes'

                                    sh 'flutter --version'
                                    sh 'flutter pub get'

                                    dir('android') {
                                        sh 'bundle config set path vendor/bundle'
                                        sh 'bundle install'

                                        if (params.BUILD_TYPE == 'DEBUG') {
                                            sh '''
                                                ARTIFACT_TYPE=APK \
                                                BUILD_TYPE=DEBUG \
                                                VERSION_NAME=${APP_VERSION_NAME} \
                                                VERSION_CODE=${APP_VERSION_CODE} \
                                                bundle exec fastlane build_android
                                            '''
                                        } else {
                                            if (params.DISTRIBUTION == 'FIREBASE') {
                                                withCredentials([
                                                    string(credentialsId: 'firebase-android-app-id', variable: 'FIREBASE_ANDROID_APP_ID'),
                                                    string(credentialsId: 'firebase-token', variable: 'FIREBASE_TOKEN')
                                                ]) {
                                                    sh '''
                                                        ARTIFACT_TYPE=APK \
                                                        BUILD_TYPE=${BUILD_TYPE} \
                                                        VERSION_NAME=${APP_VERSION_NAME} \
                                                        VERSION_CODE=${APP_VERSION_CODE} \
                                                        RELEASE_NOTES_PATH="release-notes.txt" \
                                                        bundle exec fastlane firebase_release
                                                    '''
                                                }
                                            } else if (params.DISTRIBUTION == 'STORE') {
                                                sh '''
                                                    ARTIFACT_TYPE=AAB \
                                                    BUILD_TYPE=${BUILD_TYPE} \
                                                    VERSION_NAME=${APP_VERSION_NAME} \
                                                    VERSION_CODE=${APP_VERSION_CODE} \
                                                    bundle exec fastlane build_android
                                                '''
                                            } else {
                                                sh '''
                                                    ARTIFACT_TYPE=${ANDROID_ARTIFACT} \
                                                    VERSION_NAME=${APP_VERSION_NAME} \
                                                    VERSION_CODE=${APP_VERSION_CODE} \
                                                    bundle exec fastlane build_android
                                                '''
                                            }
                                        }
                                    }

                                    sh '''
                                        mkdir -p artifacts

                                        find build/app/outputs -name "*.apk" -type f -exec cp {} artifacts/ \\; || true
                                        find build/app/outputs -name "*.aab" -type f -exec cp {} artifacts/ \\; || true

                                        ls -lh artifacts || true
                                    '''

                                    archiveArtifacts artifacts: 'artifacts/*', allowEmptyArchive: true
                                }
                            }
                        }
                    }

                    if (params.PLATFORM == 'IOS' || params.PLATFORM == 'BOTH') {
                        branches['iOS Build'] = {
                            node('mac') {
                                ws("${env.WORKSPACE}@ios") {
                                    checkout scm

                                    sh 'flutter --version'
                                    sh 'flutter pub get'

                                    if (params.BUILD_TYPE == 'DEBUG') {
                                        sh '''
                                            flutter build ios --simulator --debug
                                        '''
                                    } else {
                                        if (params.DISTRIBUTION == 'NONE') {
                                            sh '''
                                                echo "Release iOS build requires Apple signing."
                                                echo "Skipping until Apple Developer Team is configured."
                                            '''
                                        } else if (params.DISTRIBUTION == 'FIREBASE') {
                                            sh '''
                                                echo "iOS Firebase requires signed IPA."
                                                echo "You need Apple Developer Team, certificate, and provisioning profile."
                                                exit 1
                                            '''
                                        } else if (params.DISTRIBUTION == 'STORE') {
                                            sh '''
                                                echo "iOS Store/TestFlight requires Apple Developer Team and App Store Connect credentials."
                                                exit 1
                                            '''
                                        }
                                    }

                                    sh '''
                                        mkdir -p artifacts
                                        find build/ios -name "*.app" -type d -print || true
                                        find build/ios -name "*.ipa" -type f -exec cp {} artifacts/ \\; || true
                                        ls -lh artifacts || true
                                    '''

                                    archiveArtifacts artifacts: 'artifacts/*', allowEmptyArchive: true
                                }
                            }
                        }
                    }

                    parallel branches
                }
            }
        }
    }

    post {
        success {
            echo 'Pipeline completed successfully.'
        }

        failure {
            echo 'Pipeline failed.'
        }
    }
}