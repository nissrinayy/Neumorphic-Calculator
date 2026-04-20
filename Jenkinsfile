import groovy.json.JsonSlurperClassic

@NonCPS
def extractHashFromResponse(String response) {
    def matcher = response =~ /"hash"\s*:\s*"([^"]+)"/
    return matcher.find() ? matcher.group(1) : ""
}

@NonCPS
def cleanJsonString(String rawOutput) {
    int firstBrace = rawOutput.indexOf('{')
    int lastBrace  = rawOutput.lastIndexOf('}')
    if (firstBrace == -1 || lastBrace == -1) return null
    return rawOutput.substring(firstBrace, lastBrace + 1)
}

pipeline {
    agent any

    parameters {
        choice(
            name: 'BUILD_TYPE',
            choices: ['debug', 'release'],
            description: 'Tipe build APK'
        )
    }

    environment {
        ANDROID_HOME     = "C:\\Users\\Nisrina\\AppData\\Local\\Android\\Sdk"
        ANDROID_SDK_ROOT = "${ANDROID_HOME}"
        FLUTTER_HOME     = "D:\\MobDev\\Flutter SDK\\flutter"
        JAVA_HOME        = "C:\\Program Files\\Android\\Android Studio\\jbr"

        PATH = "${FLUTTER_HOME}\\bin;${JAVA_HOME}\\bin;${ANDROID_HOME}\\platform-tools;${ANDROID_HOME}\\emulator;${env.PATH}"

        AVD_NAME    = "Pixel_4_XL"
        APP_PACKAGE = "com.ragheb.neumorphic_calculator"

        MOBSF_URL   = "http://localhost:8000"
        MOBSF_TOKEN = "67f8dcdbaf63751750653685407053c3e1762a3394c5833de1d00379ca06c0fe"
    }

    stages {

        // ================= CLEAN =================
        stage('Clean Workspace') {
            steps {
                deleteDir()
            }
        }

        // ================= CHECKOUT =================
        stage('Checkout') {
            steps {
                git branch: 'main', url: 'https://github.com/nissrinayy/Neumorphic-Calculator.git'
            }
        }

        // ================= PREPARE =================
        stage('Prepare') {
            steps {
                bat 'if not exist apk-outputs mkdir apk-outputs'
            }
        }

        // ================= FLUTTER =================
        stage('Flutter Setup') {
            steps {
                bat """
                flutter clean
                flutter pub get
                flutter doctor -v
                """
            }
        }

        // ================= BUILD =================
        stage('Build APK') {
            steps {
                script {
                    def buildResult = bat(
                        script: "flutter build apk --${params.BUILD_TYPE}",
                        returnStatus: true
                    )

                    if (buildResult != 0) {
                        error "Build gagal!"
                    }

                    def apkPath = "${env.WORKSPACE}\\build\\app\\outputs\\flutter-apk\\app-${params.BUILD_TYPE}.apk"

                    if (!fileExists(apkPath)) {
                        error "APK tidak ditemukan!"
                    }

                    // VALIDASI APK PACKAGE NAME
                    echo "Checking APK package..."
                    bat "aapt dump badging \"${apkPath}\" | findstr package"

                    def timestamp = new Date().format("yyyyMMdd_HHmmss")
                    def finalApk = "${env.WORKSPACE}\\apk-outputs\\calculator-${params.BUILD_TYPE}-${timestamp}.apk"

                    bat "copy \"${apkPath}\" \"${finalApk}\""

                    env.FINAL_APK = finalApk

                    echo "APK ready: ${finalApk}"
                }
            }
        }

        // ================= EMULATOR =================
        stage('Start Emulator') {
            steps {
                bat """
                start /b "" "${env.ANDROID_HOME}\\emulator\\emulator.exe" ^
                -avd "${env.AVD_NAME}" ^
                -no-window -no-audio -gpu swiftshader_indirect -wipe-data
                """
                sleep 60
                bat "adb wait-for-device"
            }
        }

        // ================= INSTALL =================
        stage('Install APK') {
            steps {
                script {
                    echo "Cleaning old apps..."

                    bat "adb uninstall com.ragheb.neumorphic_calculator || echo not found"
                    bat "adb uninstall com.weather.app || echo not found"

                    echo "Installing APK..."

                    bat "adb install -r \"${env.FINAL_APK}\""

                    echo "Installed apps:"
                    bat "adb shell pm list packages | findstr ragheb"
                    bat "adb shell pm list packages | findstr weather"
                }
            }
        }

        // ================= SAST =================
        stage('SAST') {
            steps {
                script {
                    def upload = bat(
                        script: """
                        @curl -s ^
                        -H "Authorization: ${env.MOBSF_TOKEN}" ^
                        -F "file=@${env.FINAL_APK}" ^
                        ${env.MOBSF_URL}/api/v1/upload
                        """,
                        returnStdout: true
                    ).trim()

                    def hash = extractHashFromResponse(upload)
                    if (!hash) error "Upload gagal"

                    env.APK_HASH = hash

                    bat """
                    @curl -s -X POST ^
                    -H "Authorization: ${env.MOBSF_TOKEN}" ^
                    --data "hash=${hash}" ^
                    ${env.MOBSF_URL}/api/v1/scan
                    """
                }
            }
        }

        // ================= CLEANUP =================
        stage('Cleanup') {
            steps {
                bat 'taskkill /F /IM qemu-system-x86_64.exe /T || echo Emulator stopped'
            }
        }
    }

    post {
        always {
            archiveArtifacts artifacts: 'apk-outputs/*.apk', allowEmptyArchive: true
        }
    }
}

