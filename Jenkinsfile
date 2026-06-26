pipeline {
    agent { label 'mac' }

    parameters {
        choice(
            name: 'ARTIFACT_TYPE',
            choices: ['APK', 'AAB'],
            description: 'Choose Android artifact type to build and upload'
        )
    }

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
                sh 'java -version'
                sh 'ruby -v'
                sh 'bundle -v'
                sh 'echo "Selected artifact type: ${ARTIFACT_TYPE}"'
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

        stage('Build and Upload to Firebase') {
            steps {
                dir('android') {
                    withCredentials([
                        string(credentialsId: 'firebase-android-app-id', variable: 'FIREBASE_ANDROID_APP_ID'),
                        string(credentialsId: 'firebase-token', variable: 'FIREBASE_TOKEN')
                    ]) {
                        sh '''
                            echo "Building artifact type: ${ARTIFACT_TYPE}"
                            ARTIFACT_TYPE=${ARTIFACT_TYPE} bundle exec fastlane firebase_release
                        '''
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

                if [ "$ARTIFACT_TYPE" = "AAB" ]; then
                    ARTIFACT_FILE=$(find build/app/outputs/bundle/release -name "*.aab" -type f 2>/dev/null | head -1 || true)
                    EXT="aab"
                else
                    ARTIFACT_FILE=$(find build/app/outputs/flutter-apk -name "*.apk" -type f 2>/dev/null | head -1 || true)
                    EXT="apk"
                fi

                if [ -n "$ARTIFACT_FILE" ]; then
                    ARTIFACT_NAME="smartserve-flutter-${BUILD_NUMBER}.${EXT}"
                    cp -f "$ARTIFACT_FILE" "artifacts/$ARTIFACT_NAME"
                    cp -f "$ARTIFACT_FILE" "$HOME/Desktop/jenkins-artifacts/flutter-android/$ARTIFACT_NAME"

                    echo "Artifact copied:"
                    ls -lh artifacts/
                    ls -lh "$HOME/Desktop/jenkins-artifacts/flutter-android/"
                else
                    echo "No artifact found for ARTIFACT_TYPE=$ARTIFACT_TYPE"
                fi
            '''

            archiveArtifacts artifacts: 'artifacts/*', allowEmptyArchive: true
        }

        success {
            echo 'Flutter Android build and Firebase upload completed successfully.'
        }

        failure {
            echo 'Flutter Android build or Firebase upload failed.'
        }
    }
}