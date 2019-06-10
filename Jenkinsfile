/* A declarative groovy pipeline for one compiler and one target, trigger this from a upstream job
  Build and test the BLE examples. This script is called from a library watching a multibranch trigger,
  or on a cron nightly.
Parameters :
requested_action: build/test .
    To Test just a single example specify a basic match (ignored if requested_action=build)
    To Build only and skip all tests, specify requested_action = build
example_filtering: (string match - not a regex)
    *BLE_BatteryLev* will only build and run tests that match batterylevel
scanner_target: (defaults to target)
    Some targets run out of ram for the scanner target, so you may force suite to use a specific scanner,
    the default behaviour automatically chooses a scanner for you if the target is not listed in "valid_scanner_targets"
retry_count : 3
    You can set this to 1 or some other positive integer to change test framework retries
xxx_repo_target : ARMmbed/<xxx_repo>:master
    Allows you to overrride the target repo xxx_repo , which by default is master branch
    You can specify a tag, by just specifying the tag, example "mbed-os-5.12" or
    If the parameter contains a colon, you can point to a fork "MyUser/MyRepo:mybranch"
    example: "SomeGitUser/mbed-os-example-ble:development" will clone the SomeGitUser fork, and
    checkout their development branch.
xxx_repo_source : ""
    Pull request number this is an integer value corresponding to the PR number and will
    fetch the head of the PR if provided.
*/
import groovy.json.JsonOutput
import groovy.json.JsonSlurper
import groovy.transform.Field

// Map of hosts by target
@Field
def targets_to_host = [ // all https and port 443
        "DISCO_L475VG_IOT01A": "auli", // also ruka
        "NRF52840_DK"        : "auli", // irma only has 1
        "NRF52_DK"           : "hanna", // also irma
        "NRF51_DK"           : "hanna", // also irma
        "CY8KIT_062_BLE"     : "ronja",
        "MTB_UBLOX_ODIN_W2"  : "irma", // also ruka
        "MTB_UBLOX_NINA_B1"  : "irma",
        "MTB_ADV_WISE_1530"  : "irma",
        "MTB_LAIRD_BL600"    : "irma",
        "MTB_LAIRD_BL652"    : "irma",
        "MTB_LAIRD_BL654"    : "irma",
        "K64F"               : "mervi",  // also ruka, rauni, irma
        "K66F"               : "mervi",  // also ruka, rauni
]

// Targets that can play role of the scanner and DUT.
// if DUT target it not in this list, then the scanner needs to be chosen from this list
@Field
def valid_scanner_targets = ['NRF52_DK', 'NRF52840_DK', 'K66F', 'K64F', 'MTB_LAIRD_BL654', "MTB_UBLOX_ODIN_W2", "DISCO_L475VG_IOT01A"]
// Targets that support using a shield or do not add the feature by default on targets.json
@Field
def add_ble_feature_targets = ['K66F', 'CY8CKIT_062_BLE']

@Field
def skipped_tests = [
        '*'                : [
                // *=skip these of all targets, the * is not a regex!
                "Ble_Examples_thermometer_advertising_test"                 : "unstable",    // floating point data packing issue
                "Ble_Examples_thermometer_broadcast_payload_increasing_test": "unstable",    // floating point data packing issue
                "Ble_Examples_thermometer_broadcast_payload_reseting_test"  : "unstable",    // floating point data packing issue
                "Ble_Examples_thermometer_broadcast_payload_test"           : "unstable",    // floating point data packing issue
                "Ble_Examples_GAPButton_payload_test"                       : "unstable",    // Need to fix this one
                "Ble_Examples_beacon_broadcast_payload_test"                : "unstable",    // Need to fix this one
                "Ble_Examples_heart_rate_advertising_test"                  : "unstable",    // Need to fix this one
        ],
        'NUCLEO_F401RE'    : [
                "Ble_Examples_led_blinker_toggle_test": "unstable"
        ],
        'NRF51_DK'         : [
                "Ble_Examples_led_blinker_toggle_test": "unstable"
        ],
        'NRF52_DK'         : [
                "Ble_Examples_led_blinker_toggle_test": "unstable"
        ],
        'NRF52840_DK'      : [
                "Ble_Examples_led_blinker_toggle_test": "unstable"
        ],
        'MTB_UBLOX_NINA_B1': [
                "Ble_Examples_led_blinker_toggle_test": "unstable"
        ]
]

// simple lambda retry function , that rethrows the 1st failed invocation
@NonCPS
def retry(int times = 3,  Closure body) {
def Closure errorHandler = { echo "WARNING retry: ${e.message}"}

int retries = 0
def exceptions = []
    while(retries++ < times) {
        try {
            echo "Retry() ${retries}/${times}"
            return body.call()
        } catch(e) {
            exceptions << e
            errorHandler.call(e)
        }
    }
    throw exceptions[0]
}

// repo_name = repository name in the ARMMbed account
// target = targeted branch, if this contains a colon then the branch follows the colon and the fill repo name precedes the colonr
//          Example: target="GitUserName/mbed-os:integ"
//                   git clone git@github.com:GitUserName/mbed-os .
//                   git checkout integ
// pull_request = pull request number only
@NonCPS
def checkout_target_source(String repo_name, String target, String pull_request) {
    def repo_url = "https://github.com/ARMmbed/${repo_name}"
    if (target != null && target.contains(":")) {
        repo_url = "https://github.com/" + target.split(":")[0]
        target = target.split(":")[1]
    }
    echo "checkout_target_source() Repo root URL: ${repo_url}"

    // check out repo
    retry(3, {git url: repo_url, credentialsId: 'b40e9021-f946-4feb-ac25-5a343649e878'})

    if (target != null && target != '') {
        echo "===> TargetBranch: ${target}"
        git url: repo_url, branch: target
        sh "git checkout ${target}"  // actually switch in the environment
    }

    if (pull_request != null && pull_request != '') {
        checkout([$class                           : 'GitSCM',
                  branches                         : [[name: "FETCH_HEAD"]],
                  credentialsId                    : usernamePassword(credentialsId: 'b40e9021-f946-4feb-ac25-5a343649e878', passwordVariable: 'PASS', usernameVariable: 'USER'),
                  doGenerateSubmoduleConfigurations: false,
                  extensions                       : [[$class: 'LocalBranch']],
                  userRemoteConfigs                : [[refspec: "+refs/pull/${pull_request}/head:refs/remotes/origin/PR-${pull_request}",
                                                       url    : repo_url]]
        ])
        // Some trace for us to see what was going in
        sh "git --no-pager log -1"
    }
}

def write_json(output_json) {
    // Write each json test suite to its own file
    output_json.each { test_name, json ->
        writeFile file: test_name, text: json
        archiveArtifacts test_name
    }
}

def retrieve_available_tests(String input, String filter_opt) {
    /*
    Process the output from cliapp and extract available test
    filter_opt allows setting a way to reduce which examples we compile&run
    Using the following format:
    | 1  | Ble_Examples_battery_level_broadcast_payload_test | released | regression | cliapp | battery_level | ['BLE_BatteryLevel'] | ['BleExamples::Battery'] | ['MTB_UBLOX_NINA_B1', 'NRF51_DK', 'NRF52_DK', 'K64F', 'NUCLEO_F401RE', 'NRF51_DONGLE'] |
    Matches | * |(TEST_NAME) | * | * | (TYPE) | * | (EXAMPLE_NAMES) | * |
    Match 3 is used to determine the type of test, 'cliapp' or 'serial'
    Match 4 is a list of Example names used to select the required tests.
    */
    final test_info_matcher = /^\|\s*\d+\s*\|\s*([a-zA-Z0-9_]+)\s*\|(?:[\s\S]+?\|){2}\s*([a-zA-z0-9_']*)\s*\|(?:[\s\S]+?\|)\s*(\[[a-zA-z0-9,_ ']+\])[\s\S]+$/
    if (filter_opt != null && filter_opt != '') {
        echo "===> Filter tests on ${filter_opt}"
    }

    test_list = []
    input.split('\n').each { line ->
        if (filter_opt != null && filter_opt != '') {
            if (line ==~ filter_opt) {
            } else {
                line = "" // skip
            }
        }
        def matches = line =~ test_info_matcher
        if (matches.asBoolean()) {
            def name = matches[0][1]
            def type = matches[0][2]
            def examples = matches[0][3].replace("'", '"') //Json slurper can't handle `'`
            examples = new JsonSlurper().parseText(examples)
            test_list << [name, type, examples]
        }
    }
    return test_list
}

@NonCPS
def generate_test_suites(List available_tests, String toolchain, String target, String target_scanner, String cliapp_binary_path, String extension, String extension_scanner, Map skip_list, retry_count) {
    /*
    Generate json testsuites for each of the provided available tests for a
    particular target and target_scanner.
    */
    def testcases = [:]
    def serial_testcases = [:]

    available_tests.each { name, type, examples ->
        def testcase = ["name": name]
        if (skip_list) {
            def reason = skip_list.get(name)
            if (reason) {
                testcase['config'] = [:]
                testcase['config']['execution'] = [
                        'skip': [
                                'value' : true,
                                'reason': reason
                        ]
                ]
                echo "===> Because of ${reason}, skipping test ${name} on ${target}"
            }
        }
        // Store the different types of test separately.
        if (type == 'serial') {
            if (!serial_testcases.containsKey(examples)) {
                serial_testcases[examples] = []
            }
            serial_testcases[examples] << testcase
        } else { // Default to standard "cliapp" test
            if (!testcases.containsKey(examples)) {
                testcases[examples] = []
            }
            testcases[examples] << testcase
        }
    }

    // Used to store the json output
    def test_outputs = [:]

    // Write out standard testcases
    testcases.each { test_names, testcase ->
        test_name = test_names[0]
        def example_binary_path = "${env.WORKSPACE}/work/mbed-os-example-ble/${test_name}/BUILD/${target}/${toolchain}/${test_name}.${extension}"

        def test_suite = [
                "default": [
                        "retryCount"  : retry_count.toInteger(),
                        "retryReason" : "includeFailures",
                        "requirements": [
                                "duts": [
                                        "1": [
                                                "platform_name": target_scanner,
                                                "application"  : [
                                                        "bin": cliapp_binary_path
                                                ]
                                        ],
                                        "2": [
                                                "platform_name": target,
                                                "application"  : [
                                                        "bin"          : example_binary_path,
                                                        "baudrate"     : 9600,
                                                        "init_cli_cmds": "",
                                                        "post_cli_cmds": ""
                                                ]
                                        ]
                                ]
                        ]
                ]
        ]
        test_suite.testcases = testcase
        def test_suite_json = JsonOutput.prettyPrint(JsonOutput.toJson(test_suite))
        test_name += '.json'
        test_outputs[test_name] = test_suite_json
    }

    // Write out serial testcases
    serial_testcases.each { test_names, testcase ->
        def test_name = test_names[0]
        def support_example = test_names[1]
        def example_binary_path = "${env.WORKSPACE}/work/mbed-os-example-ble/${test_name}/BUILD/${target}/${toolchain}/${test_name}.${extension}"
        def support_binary_path = "${env.WORKSPACE}/work/mbed-os-example-ble/${support_example}/BUILD/${target}/${toolchain}/${support_example}.${extension}"
        def serial_test_suite = [
                "default": [
                        "retryCount"  : retry_count,
                        "retryReason" : "includeFailures",
                        "requirements": [
                                "duts": [
                                        "*": [
                                                "platform_name": target,
                                                "baudrate"     : 9600
                                        ],
                                        "1": [
                                                "application": [
                                                        "bin"          : example_binary_path,
                                                        "init_cli_cmds": "",  // init/post cmds not supported for * config
                                                        "post_cli_cmds": ""
                                                ]
                                        ],
                                        "2": [
                                                "application": [
                                                        "bin"          : support_binary_path,
                                                        "init_cli_cmds": "",
                                                        "post_cli_cmds": ""
                                                ]
                                        ]
                                ],
                        ]
                ]
        ]
        serial_test_suite.testcases = testcase
        def test_suite_json = JsonOutput.prettyPrint(JsonOutput.toJson(serial_test_suite))
        test_name += '.json'
        test_outputs[test_name] = test_suite_json
    }
    return test_outputs
}

@NonCPS
def find_target_extension(targets, target_name) {
    def target = targets.get(target_name)

    if (['MTB_LAIRD_BL654'].contains(target_name)) {
        return "hex"
    }
    while (target != null) {
        def extension = target.get("OUTPUT_EXT")

        if (extension) {
            return extension
        } else {
            def base_targets = target.get("inherits")
            if (base_targets) {
                // unknown rules about multiple inheritance; ignore it
                target = targets.get(base_targets[0])
            } else {
                break
            }
        }
    }
    // no extension found, return the implicit value
    return "bin"
}

// lookup in the hosts map for any scanner target on that host that has enough RAM to run the scanner role
@NonCPS
def get_scanner_target(String host_name, String target, List valid_scanner_targets, Map targets_to_host) {
    if (valid_scanner_targets.contains(target)) {
        return target
    }
    targets_to_host.each { key, value ->
        if (value == host_name && valid_scanner_targets.contains(key)) {
            return key
        }
    }
    return target // a fallback
}


// see confluence Raas instances page.
@NonCPS
def get_raas_host(String target_name, Map targets_to_host) {
    if (targets_to_host.containsKey(target_name)) {
        return targets_to_host[target_name]
    } else {
        return "irma"
    }
}

@NonCPS
def get_raas_address(target_name, Map targets_to_host) {
    return "https://${get_raas_host(target_name, targets_to_host)}.mbedcloudtesting.com:443" // "https://irma.mbedcloudtesting.com:443"
}

def build_example(example_name, target, toolchain, add_ble_feature_targets, mbed_os_source_path, extension, pyenv_path, python_logs) {
    echo "===> Build example ${example_name} for ${target} "
    dir(example_name) {
        // only targets that do not already enable BLE to be forced on /use shield
        if (add_ble_feature_targets.contains(target)) {
            // Edit the mbed_app.json to turn on the feature
            sh "find -maxdepth 2 -type f -name mbed_app.json -exec sed -i 's/K64F/${target}/g' {} +"
            sh "cp mbed_app.json mbed_app.${target}.json"
            archiveArtifacts "mbed_app.${target}.json"
        } else {
            echo "Target ${target} is assumed to have FEATURE_BLE added in mbed_app or in targets.json"
        }
        execute("ln -s ${mbed_os_source_path} mbed-os")
        sh """
            . ${pyenv_path}/bin/activate >> ${python_logs}
            mbed compile -t ${toolchain} -m ${target}
        """
        def binary_path = "BUILD/${target}/${toolchain}/${example_name}.${extension}"
        // If a future need, this is the point to do archiveArtifacts binary_path
    }
}

// returns a closure, which the caller may execute on a slave
def run_ble_test(_map) {
    return {
        def toolchain = _map.toolchain
        def target = _map.target
        def target_scanner = _map.target_scanner
        def params = _map // for code that uses the params like for SCM params.
        def RAAS_QUEUE_TIMEOUT = "300" // interval in seconds
        def cliapp_binary_path = ""
        def pyenv_path = ""
        def python_logs = ""
        def change_logs = "" // commit level
        // A single copy, which gets softlinked per project
        def mbed_os_source_path = ""
        def target_st_bluenrg_source = ""
        def target_cordio_bluenrg_source = ""

        def available_tests = []
        def extension = "" // binary filetype for the DUT
        def extension_scanner = "" // binary filetype for the scanner
        def output_json = [:]

        // no support of a post block per parallel, using a finally block to fetch logs per parallel
        try {
            stage("${_map.toolchain}_${_map.target} Pipe") {
                echo "===> Checkout ${_map.toolchain}_${_map.target}"
                dir("work") {
                    deleteDir()
                }
                echo "run_ble_test toolchain = ${_map.toolchain} target = ${_map.target}"

                dir("work") {
                    script {
                        mbed_os_source_path = "${env.WORKSPACE}/work/mbed-os"
                        echo "mbed_os_source_path = ${mbed_os_source_path}"
                        python_logs = "${env.WORKSPACE}/work/envlog.txt"
                        change_logs = "${env.WORKSPACE}/work/commitslog.txt"
                    }
                    dir("mbed-os") {
                        checkout_target_source('mbed-os.git', params.mbed_os_target, params.mbed_os_source)
                        script {
                            def raw_targets = sh script: 'cat targets/targets.json', returnStdout: true
                            def targets = new JsonSlurper().parseText(raw_targets)
                            extension = find_target_extension(targets, target)
                            // work out the scanner module, type and path
                            if (target_scanner == null || target_scanner == "") {
                                if (valid_scanner_targets.contains(target)) {
                                    target_scanner = target
                                } else {
                                    def host = get_raas_host(target, targets_to_host)
                                    target_scanner = get_scanner_target(host, target, valid_scanner_targets, targets_to_host)
                                    echo "Alternative scanner target selected: ${target_scanner}"
                                }
                            }
                            extension_scanner = find_target_extension(targets, target_scanner)
                            cliapp_binary_path = "BUILD/${target_scanner}/${toolchain}-TOOLCHAINS_PROFILE/ble-cliapp"
                        }
                    }
                    // check out the BLE shield drivers and then use a symlink in each app
                    dir("shields/TARGET_ST_BLUENRG") {
                        checkout_target_source('ble-x-nucleo-idb0xa1.git', params.bluenrg_target, params.bluenrg_source)

                        script {
                            target_st_bluenrg_source = sh script: 'pwd', returnStdout: true
                            target_st_bluenrg_source = target_st_bluenrg_source.trim()
                        }
                    }
                    dir("shields/TARGET_CORDIO_BLUENRG") {
                        checkout_target_source('cordio-ble-x-nucleo-idb0xa1.git', params.cordio_bluenrg_target, params.cordio_bluenrg_source)
                        script {
                            target_cordio_bluenrg_source = sh script: 'pwd', returnStdout: true
                            target_cordio_bluenrg_source = target_cordio_bluenrg_source.trim()
                        }
                    }
                    echo "Shared copy of mbed-os is in : [${mbed_os_source_path}]"
                    dir ("ble-cliapp") {
                        checkout_target_source('ble-cliapp.git', params.ble_cliapp_target, params.ble_cliapp_source)
                        execute("ln -s ${mbed_os_source_path} mbed-os")
                        dir("shields") {
                            execute("ln -s ${target_st_bluenrg_source} TARGET_ST_BLUENRG")
                            execute("ln -s ${target_cordio_bluenrg_source} TARGET_CORDIO_BLUENRG")
                        }
                        archiveArtifacts "toolchains_profile.json"

                        sh script: """
                            pwd > ${change_logs}
                            git log -1 --oneline >> ${change_logs}
                            cd mbed-os
                            pwd >> ${change_logs}
                            git log -1 --oneline >> ${change_logs}
                            cd ..
                            cd shields/TARGET_ST_BLUENRG
                            pwd >> ${change_logs}
                            git log -1 --oneline >> ${change_logs}
                            cd ../..
                            cd shields/TARGET_CORDIO_BLUENRG
                            pwd >> ${change_logs}
                            git log -1 --oneline >> ${change_logs}
                            cd ../..
                        """
                    }
                    archiveArtifacts "commitslog.txt"
                } // work

                dir("work") {
                    echo "===> Python environ ${_map.toolchain}_${_map.target}"
                    // Note: Dependencies were broken on some machines when using --system-site-packages flag
                    script {
                        pyenv_path = "${env.WORKSPACE}/work/pyenv"
                        echo "python environment path = ${pyenv_path}"
                    }

                    // all logs in this section go into the "${env.WORKSPACE}/work/envlog.txt" file
                    sh """
                        python -m virtualenv ${pyenv_path}  >> ${python_logs}
                    """
                    dir ('mbed-os') { // NOTE: we touch the requirements.txt file
                        sh """
                            . ${pyenv_path}/bin/activate >> ${python_logs}
                            git checkout 0063e5d -- requirements.txt
                            pip install -r requirements.txt >> ${python_logs}
                            pip install -U mbed-cli >> ${python_logs}
                            pip install idna >> ${python_logs}
                        """
                    }
                    dir('mbed-clitest'){
                        git credentialsId: 'b95cd122-dfc3-4028-b219-ad30105f0e61', url: 'git@github.com:ARMmbed/mbed-clitest.git'
                        sh """
                            . ${pyenv_path}/bin/activate >> ${python_logs}
                            git checkout e5c1f87
                            python setup.py install --force >> ${python_logs}
                        """
                    }
                    dir('raas-pyclient'){
                        git credentialsId: 'b95cd122-dfc3-4028-b219-ad30105f0e61', url: 'git@github.com:ARMmbed/raas-pyclient.git'
                        sh """
                            . ${pyenv_path}/bin/activate >> ${python_logs}
                            git checkout 03cfddc
                            python setup.py install --force >> ${python_logs}
                        """
                    }
                    dir('mbed-os-tools') {
                        git credentialsId: 'b95cd122-dfc3-4028-b219-ad30105f0e61', url: 'git@github.com:ARMmbed/mbed-os-tools.git'
                        sh "git checkout f224c48"
                        dir ('packages/mbed-host-tests/') {
                            sh """
                                . ${pyenv_path}/bin/activate >> ${python_logs}
                                python setup.py install --force >> ${python_logs}
                            """
                        }
                        sh """
                            . ${pyenv_path}/bin/activate >> ${python_logs}
                            python setup.py install --force >> ${python_logs}
                        """
                    }
                    sh """
                        . ${pyenv_path}/bin/activate >> ${python_logs}
                        pip install -U colorama==0.3.9 >> ${python_logs}
                        pip freeze >> ${python_logs}
                    """
                }

                echo "===> Build Cliapp ${_map.toolchain}_${_map.target}"
                if (params.requested_action == 'test') {
                    dir("work") {
                        dir("ble-cliapp") {
                            if (add_ble_feature_targets.contains(target)) {
                                // Edit the mbed_app.json to turn on the feature
                                sh "find -maxdepth 2 -type f -name mbed_app.json -exec sed -i 's/K64F/${target}/g' {} +"
                                sh "cp mbed_app.json cliapp_mbed_app.${target}.json"
                                archiveArtifacts "cliapp_mbed_app.${target}.json"
                            } else {
                                echo "Target ${target} is assumed to have FEATURE_BLE added in mbed_app or in targets.json"
                            }
                            // NOTE: the custom toolchain profile used here compiles for smallest possible, because cliapp is too big otherwise
                            sh """
                            . ${pyenv_path}/bin/activate >> ${python_logs}
                            mbed compile -t ${toolchain} -m ${target_scanner} --profile toolchains_profile.json
                            """
                            script {
                                cliapp_binary_path = cliapp_binary_path + "." + extension_scanner
                                // If a future need arrises, here is the point to do archiveArtifacts cliapp_binary_path
                                // build an abs path for the test suite JSON
                                cliapp_binary_path = "${env.WORKSPACE}/work/ble-cliapp/" + cliapp_binary_path
                            }
                        }
                    }
                }

                echo "===> Prepare Test list ${_map.toolchain}_${_map.target}"
                dir("work") {
                    dir("ble-tests-common.git") {
                        checkout_target_source('ble-tests-common.git', params.ble_common_target, params.ble_common_source)
                        sh """
                            . ${pyenv_path}/bin/activate >> ${python_logs}
                            pip install .
                        """
                    }

                    // Note: we ignore the jenkins checked-out copy, which contains this file we are executing.
                    dir("mbed-os-example-ble-tests") {
                        checkout_target_source('mbed-os-example-ble-tests.git', params.ble_examples_tests_target, params.ble_examples_tests_source)
                        sh "git log -3"
                        //
                        echo "Replace NRF51_DK with our target: ${target} into the test_env.py"
                        sh "find -maxdepth 2 -type f -name test_env.py -exec sed -i 's/NRF51_DK/${target}/g' {} +"
                    }

                    dir("mbed-os") {
                        script {
                            def test_list = sh script: """
                                . ${pyenv_path}/bin/activate >> ${python_logs}
                                clitest --list --tcdir ../mbed-os-example-ble-tests -vv
                            """, returnStdout: true
                            echo test_list
                            // If a filter was given, down to only the wanted name of test
                            available_tests = retrieve_available_tests(test_list, params.example_filtering)
                        }
                    } // mbed-os
                } // work

                echo "===> Example builds ${_map.toolchain}_${_map.target}"
                dir("work") {
                    dir("mbed-os-example-ble") {
                        checkout_target_source('mbed-os-example-ble.git', params.ble_examples_target, params.ble_examples_source)
                        sh """
                            pwd >> ${change_logs}
                            git log -1 --oneline >> ${change_logs}
                        """
                    }
                    archiveArtifacts "commitslog.txt"
                    dir("mbed-os-example-ble") {
                        script {
                            example_list = []
                            // When we want build only, ensure we build all folders matching BLE_.*, not just ones we will test
                            if (requested_action == 'build' || params.example_filtering == null || params.example_filtering == "") {
                                def folders = sh script: "ls -1 -d BLE_*/", returnStdout: true
                                folders.split('\n').each { folder ->
                                    folder = folder.replaceAll('/', '')
                                    example_list << folder
                                }
                            }
                            available_tests.each { test_info ->
                                (name, type, examples) = test_info
                                // Ensure all required examples are built excluding duplicates
                                examples.each { example ->
                                    if (!example_list.contains(example)) {
                                        example_list << example
                                    }
                                }
                            }

                            example_list.each { example ->
                                // symlink these shields driver repos in as well
                                dir(example) {
                                    dir("shields") {
                                        execute("ln -s ${target_st_bluenrg_source} TARGET_ST_BLUENRG")
                                        execute("ln -s ${target_cordio_bluenrg_source} TARGET_CORDIO_BLUENRG")
                                    }
                                }
                                build_example(example, target, toolchain, add_ble_feature_targets, mbed_os_source_path, extension, pyenv_path, python_logs)
                            }
                        }
                    } // mbed-os-example-ble
                } // work

                echo "===> Create suites ${_map.toolchain}_${_map.target}"
                if (params.requested_action == 'test') {
                    dir("work") {
                        dir("mbed-os") {
                            script {
                                def skipped_test = skipped_tests.get("*")
                                if (skipped_tests.get(target)) {
                                    if (null == skipped_test) {
                                        skipped_test = skipped_tests.get(target)
                                    } else {
                                        skipped_test.putAll(skipped_tests.get(target))
                                    }
                                }
                                output_json = generate_test_suites(available_tests, toolchain, target, target_scanner, cliapp_binary_path, extension, extension_scanner, skipped_test, params.retry_count)
                                write_json(output_json) // archive the json suite files
                            }
                        }
                    }
                }
                echo "===> Test ${_map.toolchain}_${_map.target}"
                if (params.requested_action == 'test') {
                    dir("work") {
                        dir("mbed-os") {
                            script {
                                // NOTE: At build time, we do not check that the scanner is always on the same raas server as DUT, but don't block build if this is not true!
                                //       Will assert if the built targets are not on the same RAAS to save things trying to run if they cannot.
                                def raas_address = get_raas_address(target, targets_to_host)
                                def raas_address_scanner = get_raas_address(target_scanner, targets_to_host)
                                assert raas_address_scanner == raas_address: "lookup RAAS for ${target} differs to server for ${target_scanner}."

                                output_json.each { test_suite, dummy ->
                                    sh """
                                        . ${pyenv_path}/bin/activate >> ${python_logs}
                                        clitest --tcdir ../mbed-os-example-ble-tests --suitedir . --suite ${test_suite} --type hardware --reset --raas ${raas_address} --raas_user user --raas_pwd user --raas_share_allocs --raas_queue --raas_queue_timeout ${RAAS_QUEUE_TIMEOUT} -vvv
                                    """

                                    publishHTML([allowMissing: false, alwaysLinkToLastBuild: false, keepAll: false, reportDir: 'log/', reportFiles: 'latest.html', reportName: "Test Report_${test_suite}"])
                                    junit 'log/latest.junit.xml'
                                } // loop
                            }
                        }
                    }
                }
            }
        } // try
        catch (Exception ex) {
            println("Test failure for...")
            println("${_map.toolchain}_${_map.target} Error: " + ex.toString())
            throw ex
        }
        finally {
            println("Archive logs for job ${_map.toolchain}_${_map.target}")

            dir("work") {
                if (pyenv_path) {
                    archiveArtifacts "envlog.txt"
                }
                if (params.requested_action == 'test') {
                    dir("mbed-os") {
                        archiveArtifacts 'log/**/*'
                    }
                }
            }
        }
    } // end closure
} // run_ble_test

def run_in_node(args_map) {
    return {
        node("all-in-one-build-slave") {
            withCredentials([usernamePassword(credentialsId: 'b40e9021-f946-4feb-ac25-5a343649e878', passwordVariable: 'PASS', usernameVariable: 'USER')]) {
                run_ble_test(args_map).call()
            }
        }
    }
}

pipeline {
    agent {
        node {
            label 'linux_greentea' // these seem easier to spin up
        }
    }
    parameters {
        string(name: 'toolchain',                   defaultValue: 'ARM', description: 'one or many of ARM,GCC_ARM,IAR : ARM = ARMC6')
        string(name: 'target',                      defaultValue: 'NRF52840_DK', description: 'many of NRF51_DK,NRF52_DK,NRF52840_DK,MTB_LAIRD_BL654 etc.')
        string(name: 'target_scanner',              defaultValue: '', description: 'Scanner (not-under-test) target, will autoselect one if omitted')
        choice(name: 'requested_action',            choices: ['test', 'build'], description: 'Only build or build&test ')
        string(name: 'example_filtering',           defaultValue: '.*Battery.*', description: 'optional. only examples that match ".*Battery.*"')
        string(name: 'retry_count',                 defaultValue: "3", description: 'Number of retries for every test')
        string(name: 'ble_examples_tests_target',   defaultValue: '', description: 'target branch, if preceeded by a colon, it specifies a repo')
        string(name: 'ble_examples_tests_source',   defaultValue: '', description: 'ARMmbed/mbed-os-example-ble-tests source PR# to pull. Numeric data only! ')
        string(name: 'mbed_os_target',              defaultValue: '', description: 'target branch *master*')
        string(name: 'mbed_os_source',              defaultValue: '', description: 'github.com:ARMmbed/mbed-os source PR#. Numeric data only! ')
        string(name: 'bluenrg_target',              defaultValue: '', description: 'target branch *master*')
        string(name: 'bluenrg_source',              defaultValue: '', description: 'github.com:ARMmbed/ble-x-nucleo-idb0xa1 source PR#. Numeric data only! ')
        string(name: 'cordio_bluenrg_target',       defaultValue: '', description: 'target branch *master*')
        string(name: 'cordio_bluenrg_source',       defaultValue: '', description: 'github.com:ARMmbed/cordio-ble-x-nucleo-idb0xa1 source PR#. Numeric data only! ')
        string(name: 'ble_cliapp_target',           defaultValue: '', description: 'target branch *master*')
        string(name: 'ble_cliapp_source',           defaultValue: '', description: 'github.com:ARMmbed/ble-cliapp source PR#. Numeric data only! ')
        string(name: 'ble_common_target',           defaultValue: '', description: 'target branch *master*')
        string(name: 'ble_common_source',           defaultValue: '', description: 'github.com:ARMmbed/ble-tests-common Sources. Numeric data only! Example: 3 PR#')
        string(name: 'ble_examples_target',         defaultValue: '', description: 'target branch, if a colon is present, the url for target is the domain+repo-name before the colon')
        string(name: 'ble_examples_source',         defaultValue: '', description: 'github.com:ARMmbed/mbed-os-example-ble Example: 220 ,to pull all changes in a PR#220. Numeric data only!')
    }

    stages {
        stage("Matrix") {
            steps{
                script {
                    def branches = [:]
                    def toolchains = toolchain
                    def targets = target
                    def numtargets
                    def numtoolchains

                    numtargets = targets.split(',')
                    numtoolchains = toolchains.split(',')
                    sh "${ecrLogin()}"
                    // matrix
                    for (toolchain in toolchains.split(',')) {
                        for (target in targets.split(',')) {
                            // map of all the non SCM related parameters
                            def argmap = [toolchain:toolchain, target:target, target_scanner:(target_scanner?target_scanner:""), requested_action:requested_action, example_filtering:(example_filtering?example_filtering:""), retry_count:retry_count]
                            params.each { k,v -> // add all the SCM parameters
                                if (k.contains("_source") || k.contains("_target")) {
                                    argmap[k] = v? v:""  // pass empty or nontrue values
                                }
                            }
                            branches["${toolchain} ${target}"] = run_in_node(argmap)
                        }
                    }
                    parallel branches
                }
            }
        }
    }
 }