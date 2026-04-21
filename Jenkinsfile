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

                    echo "Checking APK package..."
                    def AAPT = "C:\\Users\\Nisrina\\AppData\\Local\\Android\\Sdk\\build-tools\\36.0.0\\aapt.exe"

                    bat "\"${AAPT}\" dump badging \"${apkPath}\" | findstr package"

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
                -no-window -no-audio ^
                -gpu swiftshader_indirect -wipe-data
                """
                sleep 60
                bat "adb wait-for-device"
                bat "adb shell getprop sys.boot_completed"
            }
        }

        // ================= INSTALL =================
        stage('Install APK') {
            steps {
                script {
                    echo "Cleaning old apps..."

                    bat(script: "adb uninstall ${env.APP_PACKAGE}", returnStatus: true)

                    echo "Installing APK..."
                    bat "adb install -r \"${env.FINAL_APK}\""

                    echo "Installed apps:"
                    bat "adb shell pm list packages | findstr ragheb"
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

        // ================= DAST =================
        stage('DAST - Dynamic Analysis (MobSF)') {
            steps {
                script {
                    bat "adb shell input keyevent 82"
                    sleep 2

                    bat """
                    @curl -s -X POST ^
                    -H "Authorization: ${env.MOBSF_TOKEN}" ^
                    --data "hash=${env.APK_HASH}" ^
                    ${env.MOBSF_URL}/api/v1/dynamic/start_analysis
                    """

                    sleep 25

                    bat """
                    @curl -s -X POST ^
                    -H "Authorization: ${env.MOBSF_TOKEN}" ^
                    --data "hash=${env.APK_HASH}&default_hooks=api_monitor,ssl_pinning_bypass,root_bypass,debugger_check_bypass" ^
                    ${env.MOBSF_URL}/api/v1/frida/instrument
                    """

                    try {
                        bat "adb shell monkey -p ${env.APP_PACKAGE} --pct-syskeys 0 --throttle 1500 -v 200"
                    } catch (Exception e) {
                        echo "Monkey finished."
                    }

                    def tlsRaw = bat(
                        script: """
                        @curl -s -X POST ^
                        -H "Authorization: ${env.MOBSF_TOKEN}" ^
                        --data "hash=${env.APK_HASH}" ^
                        ${env.MOBSF_URL}/api/v1/android/tls_tests
                        """,
                        returnStdout: true
                    ).trim()

                    def tlsJson = cleanJsonString(tlsRaw)
                    if (tlsJson) {
                        writeFile file: 'tls_report.json', text: tlsJson
                    } else {
                        echo "⚠️ TLS JSON report could not be parsed."
                    }

                    bat """
                    @curl -s -X POST ^
                    -H "Authorization: ${env.MOBSF_TOKEN}" ^
                    --data "hash=${env.APK_HASH}" ^
                    ${env.MOBSF_URL}/api/v1/dynamic/stop_analysis
                    """

                    def raw = bat(
                        script: """
                        @curl -s -X POST ^
                        -H "Authorization: ${env.MOBSF_TOKEN}" ^
                        --data "hash=${env.APK_HASH}" ^
                        ${env.MOBSF_URL}/api/v1/dynamic/report_json
                        """,
                        returnStdout: true
                    ).trim()

                    def json = cleanJsonString(raw)
                    if (json) {
                        writeFile file: 'dast_report.json', text: json
                        archiveArtifacts artifacts: 'dast_report.json, tls_report.json', allowEmptyArchive: true
                        echo "✅ DAST Report URL: ${env.MOBSF_URL}/dynamic_report/${env.APK_HASH}/"
                    } else {
                        echo "⚠️ DAST JSON report could not be parsed."
                    }
                }
            }
        }

        // ================= CLEANUP =================
        stage('Cleanup') {
            steps {
                bat 'taskkill /F /IM qemu-system-x86_64.exe /T || echo Emulator already stopped'
            }
        }
    }

    post {
        always {
            archiveArtifacts artifacts: 'apk-outputs/*.apk', allowEmptyArchive: true
        }
    }
}