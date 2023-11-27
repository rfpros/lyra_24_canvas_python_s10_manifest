pipeline {
    agent {
        label 'USCOLOBH06'
    }

    options {
        skipDefaultCheckout(true)
        timestamps()
        disableConcurrentBuilds()
    }

    triggers {
        pollSCM 'H/15 * * * *'
    }

    environment {
        // Set to URL of first git remote config from Multibranch Pipeline definition in Jenkins
        GIT_REPO = scm.getUserRemoteConfigs()[0].getUrl()

        // Directory the west manifest is fetched into.
        // This has to match the directory in the self:path: field in the yaml file
        // As we have a chicken / egg situation if it's put in a project specific place.
        // This works for the current project but may need to be revisited if a more
        // complex scheme is used.
        MANIFEST_DIR = "west_manifest"

        // Fileshare paths for final output
        FILESHARE_SERVER = "fileshare@files.devops.rfpros.com"

        // SS script files
        SIMPLICITY_STUDIO_SCRIPT_UTIL = "C:\\SiliconLabs\\SimplicityStudio\\v5\\developer\\scripting\\runScript.bat"
        SIMPLICITY_STUDIO_SCRIPT = "${env.WORKSPACE}\\an1121-headless-builds-simplicity-studio\\PythonScripts\\ImportExistingProjectAndBuild.py"

        // Path to the ouptput
        SIMPLICITY_STUDIO_OUTPUT = "${env.WORKSPACE}\\output"

        // Version paths for main app and SDK bluetooth version
        APP_VERSION_FILE = "app_version.h"
    }

    stages {
        // Clean up the workspace
        stage('Conditional Cleanup') {
            stages {
                stage('Workspace Cleanup') {
                    steps {
                        cleanWs()
                    }
                }
                stage('Python Environment Cleanup') {
                    when {
                        expression { params.CLEAN_PYTHON_VENV }
                    }
                    steps {
                        bat """rmdir /s /q .venv"""
                    }
                }
            }
        }

        // Initialise the python environment so west will work
        stage('Init Environment') {
            steps {
                winInitPythonVirtualEnv()
                winPipInstall(name: 'pip')
                winPipInstall(name: 'rbtools')
                winPipInstall(name: 'west')
                winPipInstall(name: 'ecdsa')
            }
        }

        // Use west to pull the files to build
        stage('Checkout') {
            steps {
                // Simplicity Studio Helper Scripts
                checkout([
                    $class: 'GitSCM',
                    branches: [[name: 'main']],
                    userRemoteConfigs: [[url: 'git@git.devops.rfpros.com:silabs/an1121-headless-builds-simplicity-studio.git']],
                    extensions: [
                        [$class: 'RelativeTargetDirectory', relativeTargetDir: 'an1121-headless-builds-simplicity-studio']
                    ]
                ])

                // Ensure output dir exists
                bat """if not exist "${env.SIMPLICITY_STUDIO_OUTPUT}" mkdir "${env.SIMPLICITY_STUDIO_OUTPUT}" """

                winWestInit(
                    manifestUrl: env.GIT_REPO,
                    manifestRevision: env.BRANCH_NAME
                )

                // Fetch all supporting repositories
                winWestUpdate()
            }
        }

        stage('Environment Update') {
            steps {
                script {
                    def westConfig = readYaml file: "${MANIFEST_DIR}\\west.yml"
                    def project = westConfig['jenkins']['project']
                    def application = westConfig['jenkins']['application']
                    def module = westConfig['jenkins']['module']
                    def power = westConfig['jenkins']['power']
                    def sdk = westConfig['jenkins']['sdk']

                    // Environment variables used by firmware major regex and tagging
                    env.PROJECT = "${project.toUpperCase()}"
                    env.APPLICATION = "${application.toUpperCase()}"
                    env.MODULE = "${module.toUpperCase()}"
                    env.POWER = power

                    // Set the directory for the whole 'project' the project and modules were be created inside
                    env.PROJECT_DIR = "${project}_${application}_${module}${power}_project"

                    // Fileshare paths for final output
                    env.FILESHARE_PATH = "/var/www/html/builds/simplicity-studio/${project}"

                    // Path of the SS project
                    env.SIMPLICITY_STUDIO_PROJECT = "${env.WORKSPACE}\\${env.PROJECT_DIR}"

                    // Version paths for main app and SDK bluetooth version
                    env.APP_VERSION_PATH = "${env.SIMPLICITY_STUDIO_PROJECT}\\${env.APP_VERSION_FILE}"
                    env.BT_VERSION_FILE = "gecko_sdk_${sdk}\\protocol\\bluetooth\\inc\\sl_bt_version.h"
                    env.BT_VERSION_PATH = "${env.SIMPLICITY_STUDIO_PROJECT}\\${env.BT_VERSION_FILE}"

                    // Part numbers
                    env.SOFTWARE_PART_NUMBER = westConfig['jenkins']['part_number']
                    env.SOFTWARE_PART_NUMBER_UART = westConfig['jenkins']['part_number_uart_dfu']
                    env.SOFTWARE_PART_NUMBER_OTA = westConfig['jenkins']['part_number_ota_dfu']

                    // Project build to use
                    env.SIMPLICITY_STUDIO_BUILD_CFG = "${env.PROJECT_DIR}_release"

                    // Where to put build files
                    env.SIMPLICITY_STUDIO_CI_WORKSPACE = "${env.WORKSPACE}\\ws"
                }
            }
        }

        stage('Remove artifacts') {
            steps {
                bat """rmdir /s /q "${SIMPLICITY_STUDIO_OUTPUT}" """
                bat """mkdir "${SIMPLICITY_STUDIO_OUTPUT}" """
            }
        }

        stage('Manifest snapshot') {
            steps {
                // Freeze the manifest and save it as an artifact so we can see exactly what software was used to make the build
                bat """call $WORKSPACE\\.venv\\Scripts\\activate.bat
                       call west manifest --freeze -o $SIMPLICITY_STUDIO_OUTPUT\\west.freeze.yml
                    """
            }
        }

        stage('Build Configuration') {
            steps {
                script {
                    def fwVersionMajor
                    def fwVersionMinor
                    def fwVersionFix
                    def fwVersionSuffix

                    echo env.APP_VERSION_PATH
                    echo env.BT_VERSION_PATH

                    def app_version_lines = readFile(env.APP_VERSION_PATH).split("\n")
                    def bt_version_lines = readFile(env.BT_VERSION_PATH).split("\n")

                    // loop over app version file lines
                    app_version_lines.eachWithIndex { val, idx ->
                        // Store version major
                        if (val =~ "define[ ]+${env.PROJECT}_${env.MODULE}_${env.POWER}DB_PRODUCT_ID") {
                            def list = val.findAll("\\d+")
                            fwVersionMajor = list[list.size() - 1]
                            echo 'Found ${env.PROJECT}_${env.MODULE}_${env.POWER}DB_PRODUCT_ID'
                        }

                        // Get version fixed
                        if (val =~ /define[ ]+APP_PROPERTIES_VERSION/) {
                            fwVersionFix = val.findAll("\\d+")[0]
                          echo 'Found APP_PROPERTIES_VERSION'
                        }

                        // update line with build number or epoch time depending on branch
                        if (val =~ /define[ ]+APP_BUILD_NUMBER/) {
                            if (env.BRANCH_NAME == "main") {
                                app_version_lines[idx] = ("#define APP_BUILD_NUMBER ${env.BUILD_NUMBER}").toString()
                                echo 'Updated main build number'
                            } else {
                                app_version_lines[idx] = ("#define APP_BUILD_NUMBER ${currentBuild.startTimeInMillis.intdiv(1000)}").toString()
                            }
                        }
                    }

                    // loop over BT version lines to extract version component
                    bt_version_lines.eachWithIndex { val, idx ->
                        if (val =~ /define[ ]+SL_BT_VERSION_MAJOR/) {
                            fwVersionMinor = val.findAll("\\d+")[0]
                        }
                    }

                    // update app version file with build number modification
                    writeFile(file: env.APP_VERSION_PATH, text: app_version_lines.join('\n'))
                    echo 'version file written'

                    // Use build number instead of epoch time for main branch
                    if (env.BRANCH_NAME == "main") {
                        fwVersionSuffix = "${fwVersionMinor}.${fwVersionFix}.${env.BUILD_NUMBER}"
                    } else {
                        fwVersionSuffix = "${fwVersionMinor}.${fwVersionFix}.${currentBuild.startTimeInMillis.intdiv(1000)}"
                    }

                    env.TAG_NAME = "${env.PROJECT}-${env.APPLICATION}-REL-X.${fwVersionSuffix}"
                    echo "TAG NAME: " + env.TAG_NAME

                    // Update Jenkins UI with extracted version
                    currentBuild.displayName = fwVersionSuffix

                    //
                    // Additional Environment Variables
                    //
                    env.VERSION = "${fwVersionMajor}.${fwVersionSuffix}"

                    env.RESULT_PREFIX = "${env.SIMPLICITY_STUDIO_CI_WORKSPACE}\\${env.PROJECT_DIR}\\${env.SIMPLICITY_STUDIO_BUILD_CFG}\\${env.PROJECT_DIR}"
                    env.RESULT_PATH_AXF = "${env.RESULT_PREFIX}.axf"
                    env.RESULT_PATH_BIN = "${env.RESULT_PREFIX}.bin"
                    env.RESULT_PATH_HEX = "${env.RESULT_PREFIX}.hex"
                    env.RESULT_PATH_S37 = "${env.RESULT_PREFIX}.s37"

                    env.GBL_PREFIX = "${env.SIMPLICITY_STUDIO_CI_WORKSPACE}\\${env.PROJECT_DIR}\\output_gbl"
                    env.GBL_PATH_FULL = "${env.GBL_PREFIX}\\full-crc.gbl"
                    env.GBL_PATH_APP = "${env.GBL_PREFIX}\\application-crc.gbl"

                    env.OUTPUT_PREFIX = "${env.SOFTWARE_PART_NUMBER}-R${env.VERSION}"
                    env.OUTPUT_PATH = "${env.SIMPLICITY_STUDIO_OUTPUT}\\${env.OUTPUT_PREFIX}"
                    env.OUTPUT_PREFIX_UART = "${env.SOFTWARE_PART_NUMBER_UART}-R${env.VERSION}"
                    env.OUTPUT_PATH_UART = "${env.SIMPLICITY_STUDIO_OUTPUT}\\${env.OUTPUT_PREFIX_UART}"
                    env.OUTPUT_PREFIX_OTA = "${env.SOFTWARE_PART_NUMBER_OTA}-R${env.VERSION}"
                    env.OUTPUT_PATH_OTA = "${env.SIMPLICITY_STUDIO_OUTPUT}\\${env.OUTPUT_PREFIX_OTA}"

                    env.OUTPUT_PATH_WEST_FREEZE = "${env.SIMPLICITY_STUDIO_OUTPUT}\\west.freeze.yml"
                    env.OUTPUT_NAME_WEST_FREEZE = "${env.SOFTWARE_PART_NUMBER}-R${env.VERSION}_WEST.yml"

                    env.OUTPUT_FILE_AXF = "${env.OUTPUT_PREFIX}.axf"
                    env.OUTPUT_PATH_AXF = "${env.OUTPUT_PATH}.axf"
                    env.OUTPUT_FILE_BIN = "${env.OUTPUT_PREFIX}.bin"
                    env.OUTPUT_PATH_BIN = "${env.OUTPUT_PATH}.bin"
                    env.OUTPUT_FILE_HEX = "${env.OUTPUT_PREFIX}.hex"
                    env.OUTPUT_PATH_HEX = "${env.OUTPUT_PATH}.hex"
                    env.OUTPUT_FILE_S37 = "${env.OUTPUT_PREFIX}.s37"
                    env.OUTPUT_PATH_S37 = "${env.OUTPUT_PATH}.s37"
                    env.OUTPUT_FILE_ZIP = "${env.OUTPUT_PREFIX}.zip"
                    env.OUTPUT_PATH_ZIP = "${env.OUTPUT_PATH}.zip"
                    env.OUTPUT_FILE_GBL_FULL = "${env.OUTPUT_PREFIX_UART}_UART.gbl"
                    env.OUTPUT_PATH_GBL_FULL = "${env.OUTPUT_PATH_UART}_UART.gbl"
                    env.OUTPUT_FILE_GBL_APP = "${env.OUTPUT_PREFIX_OTA}_OTA.gbl"
                    env.OUTPUT_PATH_GBL_APP = "${env.OUTPUT_PATH_OTA}_OTA.gbl"
                }
            }
        }

        stage('Tag') {
            when {
                branch 'main'
            }
            steps {
                dir(MANIFEST_DIR) {
                    bat """git tag -a "${env.TAG_NAME}" -m "${env.TAG_NAME}" """
                    bat """git push origin "${env.TAG_NAME}" --follow-tags """
                }
            }
        }

        // No longer multiple builds so no paralell or encomapsing stage / stages
        stage('Build') {
            steps{

                // Pre compile must be run on the original directory so the generated files are copied and included in the
                // build
                bat """
                PUSHD .
                cd ${env.SIMPLICITY_STUDIO_PROJECT}
                CALL modules\\lyra_24_micropython_common_board\\jenkins_pre_compile.bat
                POPD
                """

                // Run build
                bat """mkdir "${env.SIMPLICITY_STUDIO_CI_WORKSPACE}" """
                bat """${env.SIMPLICITY_STUDIO_SCRIPT_UTIL} -data ${env.SIMPLICITY_STUDIO_CI_WORKSPACE} ${env.SIMPLICITY_STUDIO_SCRIPT} ${env.SIMPLICITY_STUDIO_PROJECT} --AutoEnable --buildConfig "${env.SIMPLICITY_STUDIO_BUILD_CFG}" """

                dir("${env.SIMPLICITY_STUDIO_CI_WORKSPACE}\\${env.PROJECT_DIR}") {
                    bat """create_bl_files.bat"""
                }

                // rename artifacts
                bat """copy "${env.RESULT_PATH_AXF}" "${env.OUTPUT_PATH_AXF}" """
                bat """copy "${env.RESULT_PATH_BIN}" "${env.OUTPUT_PATH_BIN}" """
                bat """copy "${env.RESULT_PATH_HEX}" "${env.OUTPUT_PATH_HEX}" """
                bat """copy "${env.RESULT_PATH_S37}" "${env.OUTPUT_PATH_S37}" """
                bat """copy "${env.GBL_PATH_FULL}" "${env.OUTPUT_PATH_GBL_FULL}" """
                bat """copy "${env.GBL_PATH_APP}" "${env.OUTPUT_PATH_GBL_APP}" """
                bat """ren "${env.OUTPUT_PATH_WEST_FREEZE}" "${env.OUTPUT_NAME_WEST_FREEZE}" """

                // archive artifacts and fail if not found
                dir(env.SIMPLICITY_STUDIO_OUTPUT) {
                    archiveArtifacts(allowEmptyArchive: false, artifacts: env.OUTPUT_FILE_AXF)
                    archiveArtifacts(allowEmptyArchive: false, artifacts: env.OUTPUT_FILE_BIN)
                    archiveArtifacts(allowEmptyArchive: false, artifacts: env.OUTPUT_FILE_HEX)
                    archiveArtifacts(allowEmptyArchive: false, artifacts: env.OUTPUT_FILE_S37)
                    archiveArtifacts(allowEmptyArchive: false, artifacts: env.OUTPUT_FILE_GBL_FULL)
                    archiveArtifacts(allowEmptyArchive: false, artifacts: env.OUTPUT_FILE_GBL_APP)
                    archiveArtifacts(allowEmptyArchive: false, artifacts: env.OUTPUT_NAME_WEST_FREEZE)
                }
            }
        }

        stage('Package') {
            steps {
                // zips files for each variant and archives them
                dir(env.SIMPLICITY_STUDIO_OUTPUT) {
                    zip dir: '',
                        glob: "${env.OUTPUT_PREFIX}*",
                        zipFile: "${env.OUTPUT_FILE_ZIP}"

                    archiveArtifacts(allowEmptyArchive: false, artifacts: env.OUTPUT_FILE_ZIP)
                }
            }
        }

        stage('Upload') {
            when {
                branch 'main'
            }
            steps {
                // when on main branch, upload zips to the file share.
                dir(env.SIMPLICITY_STUDIO_OUTPUT) {
                    bat """ssh ${env.FILESHARE_SERVER} "mkdir -p ${env.FILESHARE_PATH}" """
                    bat """scp -p ${env.OUTPUT_FILE_ZIP} fileshare@files.devops.rfpros.com:${env.FILESHARE_PATH} """
                }
            }
        }
    }
}

void winPipInstall(Map m) {
    if (!m.containsKey('name')) {
        error("name not specified in call to pipInstall")
    }
    bat """
        call "$WORKSPACE\\.venv\\Scripts\\activate.bat"
        call python311 -m pip install -U ${m['name']}
        """
}

void  winInitPythonVirtualEnv() {
    bat """
    call python311 -m venv .venv
    """
}

void winWestInit(Map m) {
    if (!m.containsKey('manifestUrl')) {
        error("manifestUrl not specified in call to westInit")
    }

    if (!m.containsKey('manifestRevision')) {
        error("manifestRevision not specified in call to westInit")
    }

    bat """
        call "$WORKSPACE\\.venv\\Scripts\\activate.bat"
        call west init -m ${m['manifestUrl']} --mr ${m['manifestRevision']}
    """
}

void winWestUpdate() {
    bat """
        call "$WORKSPACE\\.venv\\Scripts\\activate.bat"
        cd "${MANIFEST_DIR}"
        call west update
    """
}