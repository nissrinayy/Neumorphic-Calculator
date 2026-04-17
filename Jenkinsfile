import groovy.json.JsonSlurperClassic

// ================= HELPER METHODS =================
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

// ================= PIPELINE =================
pipeline {
    agent any

    parameters {
        choice(
            name: 'BUILD_TYPE',
            choices: ['debug', 'release'],
            description: 'Pilih tipe build APK (Release direkomendasikan untuk audit)'
        )
    }

    environment {
        ANDROID_HOME     = "C:\\Users\\Nisrina\\AppData\\Local\\Android\\Sdk"
        ANDROID_SDK_ROOT = "${ANDROID_HOME}"
        FLUTTER_HOME     = "D:\\MobDev\\Flutter SDK\\flutter"
        JAVA_HOME = "C:\\Program Files\\Android\\Android Studio\\jbr"
        PATH = "${FLUTTER_HOME}\\bin;${JAVA_HOME}\\bin;${ANDROID_HOME}\\platform-tools;${ANDROID_HOME}\\emulator;${env.PATH}"

        AVD_NAME    = "Pixel_4_XL"
        APP_PACKAGE = "com.ragheb.neumorphic_calculator"

        MOBSF_URL   = "http://localhost:8000"
        MOBSF_TOKEN = "67f8dcdbaf63751750653685407053c3e1762a3394c5833de1d00379ca06c0fe"
    }

    stages {

        // ================= CHECKOUT =================
        stage('Checkout') {
            steps {
                git branch: 'main', url: 'https://github.com/nissrinayy/Rain.git'
            }
        }

        // ================= PREPARE =================
        stage('Prepare Workspace') {
            steps {
                bat 'if not exist apk-outputs mkdir apk-outputs'
            }
        }

        // ================= FLUTTER ENV =================
        stage('Flutter Environment Check') {
            steps {
                bat """
                git config --global --add safe.directory "%WORKSPACE%"
                flutter config --jdk-dir "${env.JAVA_HOME}"
                flutter doctor -v
                """
            }
        }

        // ================= BUILD APK =================
        stage('Build APK') {
            steps {
                script {
                    // Clean previous build artifacts
                    bat "if exist build\\app\\outputs\\flutter-apk rmdir /s /q build\\app\\outputs\\flutter-apk"

                    // Run flutter pub get and check exit code
                    def pubGet = bat(script: 'flutter pub get', returnStatus: true)
                    if (pubGet != 0) {
                        error "flutter pub get failed with exit code: ${pubGet}"
                    }

                    // Run flutter build and check exit code
                    def buildResult = bat(script: "flutter build apk --${params.BUILD_TYPE} --verbose", returnStatus: true)
                    if (buildResult != 0) {
                        error "flutter build apk --${params.BUILD_TYPE} failed with exit code: ${buildResult}"
                    }

                    // Debug: show full build output directory tree
                    echo "=== Listing build output directory ==="
                    bat "if exist build\\app\\outputs dir /s /b build\\app\\outputs || echo build\\app\\outputs does not exist"

                    // Try standard path
                    def apkFullPath = "${env.WORKSPACE}\\build\\app\\outputs\\flutter-apk\\app-${params.BUILD_TYPE}.apk"

                    // If not found, search entire build folder for any APK
                    if (!fileExists(apkFullPath)) {
                        echo "APK not found at standard path, searching entire workspace for APKs..."
                        bat "dir /s /b \"${env.WORKSPACE}\\build\\*.apk\" || echo No APK files found anywhere in build folder"
                        error "APK not found at: ${apkFullPath} — check the search results above to find where it was placed."
                    }

                    echo "✅ APK successfully built at: ${apkFullPath}"
                }
            }
        }

        // ================= SAST =================
        stage('SAST - Static Analysis (MobSF)') {
            steps {
                script {
                    def apkRelPath  = "build\\app\\outputs\\flutter-apk\\app-${params.BUILD_TYPE}.apk"
                    def apkFullPath = "${env.WORKSPACE}\\${apkRelPath}"

                    // Debug: list directory to confirm APK exists
                    bat "dir build\\app\\outputs\\flutter-apk\\ || echo Directory not found"

                    if (!fileExists(apkFullPath)) {
                        error "APK not found: ${apkFullPath}"
                    }

                    echo "Uploading APK to MobSF (SAST)..."

                    def uploadResponse = bat(
                        script: """
                        @curl -s ^
                        -H "Authorization: ${env.MOBSF_TOKEN}" ^
                        -F "file=@${apkFullPath}" ^
                        ${env.MOBSF_URL}/api/v1/upload
                        """,
                        returnStdout: true
                    ).trim()

                    def apkHash = extractHashFromResponse(uploadResponse)
                    if (!apkHash) error "MobSF upload failed — no hash returned. Response: ${uploadResponse}"

                    env.APK_HASH = apkHash
                    echo "SAST Hash: ${apkHash}"

                    bat """
                    @curl -s -X POST ^
                    -H "Authorization: ${env.MOBSF_TOKEN}" ^
                    --data "hash=${apkHash}" ^
                    ${env.MOBSF_URL}/api/v1/scan
                    """

                    def rawReport = bat(
                        script: """
                        @curl -s -X POST ^
                        -H "Authorization: ${env.MOBSF_TOKEN}" ^
                        --data "hash=${apkHash}" ^
                        ${env.MOBSF_URL}/api/v1/report_json
                        """,
                        returnStdout: true
                    ).trim()

                    def json = cleanJsonString(rawReport)
                    if (json) {
                        writeFile file: 'sast_report.json', text: json
                        archiveArtifacts artifacts: 'sast_report.json'
                        echo "✅ SAST Report URL: ${env.MOBSF_URL}/static_analyzer/${apkHash}/"
                    } else {
                        echo "⚠️ SAST JSON report could not be parsed."
                    }
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

        // ================= INSTALL APK =================
        stage('Install APK') {
            steps {
                script {
                    def timestamp  = new Date().format("dd-MM-yyyy_HH-mm-ss")
                    def sourcePath = "${env.WORKSPACE}\\build\\app\\outputs\\flutter-apk\\app-${params.BUILD_TYPE}.apk"
                    def destPath   = "${env.WORKSPACE}\\apk-outputs\\todo-${params.BUILD_TYPE}-${timestamp}.apk"

                    bat "copy \"${sourcePath}\" \"${destPath}\""

                    bat(script: "adb uninstall ${env.APP_PACKAGE}", returnStatus: true)
                    bat "adb install -r \"${destPath}\""
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

        // ================= DOWNLOAD PDF =================
        stage('Download PDF Report') {
            steps {
                script {
                    if (env.APK_HASH) {
                        echo "Downloading SAST PDF..."
                        bat """
                        @curl -s -X POST ^
                        -H "Authorization: ${env.MOBSF_TOKEN}" ^
                        --data "hash=${env.APK_HASH}" ^
                        ${env.MOBSF_URL}/api/v1/download_pdf ^
                        -o sast_report.pdf
                        """

                        echo "Downloading DAST PDF..."
                        bat """
                        @curl -L -f ^
                        ${env.MOBSF_URL}/dynamic_pdf/${env.APK_HASH}/ ^
                        --output dast_report.pdf --silent --show-error
                        """

                        sleep 2

                        bat "dir dast_report.pdf"
                        bat "for %%I in (dast_report.pdf) do @echo Size: %%~zI bytes"

                        def size = bat(
                            script: '@for %%I in (dast_report.pdf) do @echo %%~zI',
                            returnStdout: true
                        ).trim().split("\\r?\\n")[-1]

                        if (size.toInteger() < 50000) {
                            error "DAST PDF INVALID: ${size} bytes"
                        }

                        echo "✅ PDFs downloaded successfully."
                    } else {
                        echo "⚠️ APK_HASH not found. Skipping PDF download."
                    }
                }
            }
        }

        // ================= SEND EMAIL =================
        stage('Send Email Manual') {
            steps {
                writeFile file: 'send_email.ps1', text: '''
[System.Net.ServicePointManager]::SecurityProtocol = [System.Net.SecurityProtocolType]::Tls12

$smtpServer = "smtp.gmail.com"
$smtpPort = 587
$from = "syifaamaraqueen@gmail.com"
$to = "syifaamdh@gmail.com"
$password = "lyftygsebuyndied"

$mail = New-Object System.Net.Mail.MailMessage
$mail.From = $from
$mail.To.Add($to)
$mail.Subject = "Security Report Jenkins"
$mail.Body = "Laporan PDF terlampir"

$att1 = New-Object System.Net.Mail.Attachment("sast_report.pdf")
$att2 = New-Object System.Net.Mail.Attachment("dast_report.pdf")

$mail.Attachments.Add($att1)
$mail.Attachments.Add($att2)

$smtp = New-Object System.Net.Mail.SmtpClient($smtpServer, $smtpPort)
$smtp.EnableSsl = $true
$smtp.UseDefaultCredentials = $false
$smtp.Credentials = New-Object System.Net.NetworkCredential($from, $password)
$smtp.DeliveryMethod = [System.Net.Mail.SmtpDeliveryMethod]::Network
$smtp.Timeout = 30000

$smtp.Send($mail)
'''
                bat 'powershell -ExecutionPolicy Bypass -File send_email.ps1'
            }
        }

    }

    post {
        always {
            archiveArtifacts artifacts: 'apk-outputs/*.apk', allowEmptyArchive: true
            archiveArtifacts artifacts: '*.pdf', allowEmptyArchive: true
        }
    }
}