pipeline {
    agent { label 'mac' }

    environment {
        FLUTTER_HOME = '/Users/enz/development/flutter'
        ANDROID_HOME = '/Users/enz/Library/Android/sdk'
        ANDROID_SDK_ROOT = '/Users/enz/Library/Android/sdk'
        PATH = "${FLUTTER_HOME}/bin:/opt/homebrew/bin:${ANDROID_HOME}/cmdline-tools/latest/bin:${ANDROID_HOME}/platform-tools:${env.PATH}"
    }

    stages {
        stage('Check Environment') {
            steps {
                sh 'pwd'
                sh 'which flutter'
                sh 'flutter --version'
                sh 'flutter doctor'
                sh 'java -version'
                sh 'ruby -v'
                sh 'bundle -v'
            }
        }

        stage('Flutter Dependencies') {
            steps {
                sh 'flutter pub get'
            }
        }

        stage('Install Fastlane Dependencies') {
            steps {
                dir('android') {
                    sh 'bundle config set path vendor/bundle'
                    sh 'bundle install'
                }
            }
        }

        stage('Build APK and Upload to Firebase') {
            steps {
                dir('android') {
                    withCredentials([
                        string(credentialsId: 'firebase-android-app-id', variable: 'FIREBASE_ANDROID_APP_ID'),
                        string(credentialsId: 'firebase-token', variable: 'FIREBASE_TOKEN')
                    ]) {
                        sh 'bundle exec fastlane firebase_release'
                    }
                }
            }
        }
    }

    post {
        always {
            sh '''
                mkdir -p artifacts
                mkdir -p "$HOME/Desktop/jenkins-artifacts/flutter-android"

                APK_FILE=$(find build/app/outputs/flutter-apk -name "*.apk" -type f 2>/dev/null | head -1 || true)

                if [ -n "$APK_FILE" ]; then
                    APK_NAME="smartserve-flutter-${BUILD_NUMBER}.apk"
                    cp -f "$APK_FILE" "artifacts/$APK_NAME"
                    cp -f "$APK_FILE" "$HOME/Desktop/jenkins-artifacts/flutter-android/$APK_NAME"

                    echo "APK copied:"
                    ls -lh artifacts/
                    ls -lh "$HOME/Desktop/jenkins-artifacts/flutter-android/"
                else
                    echo "No APK file found."
                fi
            '''

            archiveArtifacts artifacts: 'artifacts/*.apk', allowEmptyArchive: true
        }
    }
}