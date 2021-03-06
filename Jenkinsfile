#!/usr/bin/env groovy

properties([
        disableConcurrentBuilds()
])

// Days between clean builds
days = 10

devices = []
devices += ['xilinx_vcu1525_dynamic_6_0']
devices += ['xilinx_u250_xdma_201830_2']
devices += ['xilinx_u200_xdma_201830_2']

env.VERSION = '2018.3'

precheck_status = 'FAILURE'
sw_status = 'FAILURE'
hw_status = 'FAILURE'
setupString = """
. /tools/dist/public/modules/lnx/wrapper_bash_new.sh
module load xilinx/ta/${VERSION}_daily_latest
. /proj/xbuilds/${VERSION}_daily_latest/xbb/xrt/packages/setenv.sh
"""

def pre_check() {
  return { ->
    sh """
${setupString}
./utility/check_license.sh LICENSE.txt
./utility/check_readme.sh
"""
  }
}

def setupExample(dir, workdir) {
  return { ->
    sh """#!/bin/bash -e
${setupString}
cd ${workdir}/${dir}

echo
echo "-----------------------------------------------"
echo "PWD: \$(pwd)"
echo "-----------------------------------------------"
echo

rsync -rL \$XILINX_SDX/lnx64/tools/opencv/ lib/

make clean exe
"""
  }
}

def buildExample(target, dir, device, workdir) {
  return { ->
    if (target == "sw_emu") {
      cores = 1
      mem = 4000
      queue = "medium"
      mins = 5
    } else {
      cores = 8
      mem = 32000
      queue = "long"
      mins = 12 * 60
    }

    retry(3) {
      sh """#!/bin/bash -e
cd ${workdir}

set +e
python ${workdir}/utility/check_target_device.py ${workdir}/${dir}/description.json ${target} ${device}
rc=\$?
set -e

if [[ \$rc == 0 ]]; then
  echo "Nothing to do"
  exit 0
fi

cd ${workdir}/${dir}

echo
echo "-----------------------------------------------"
echo "PWD: \$(pwd)"
echo "-----------------------------------------------"
echo

${setupString}
export PLATFORM_REPO_PATHS=/proj/xbuilds/\${VERSION}_daily_latest/xbb/dsadev/opt/xilinx/platforms:\$PLATFORM_REPO_PATHS

# Check if a rebuild is necessary
set -x
set +e
make -q TARGETS=${target} DEVICES=\"${device}\" all
rc=\$?
set -e

# re build if required
if [[ \$rc != 0 ]]; then
export TMPDIR=\$(mktemp -d -p \$(pwd))
bsub -W ${mins} -I -q ${queue} -R "osdistro=rhel && osver==ws7" -n ${cores} -R "rusage[mem=${mem}] span[ptile=${cores}]" -J "\$(basename ${dir})-${target}" <<EOF
#!/bin/bash -ex
export TMPDIR=\$TMPDIR
cd ${workdir}/${dir}
make -j TARGETS=${target} DEVICES=\"${device}\" all
rm -rf \$TMPDIR
EOF
fi
set +x
"""
    }
  }
}

def dirsafe(device) {
  return device.replaceAll(":", "_").replaceAll("\\.", "_")
}

def runExample(target, dir, device, workdir) {
  return { ->
    if (target == "sw_emu") {
      cores = 4
      mem = 32000
      queue = "medium"
      mins = 15
    } else {
      cores = 1
      mem = 4000
      queue = "long"
      mins = 8 * 60
    }

    devdir = dirsafe(device)

    retry(3) {
      //	lock("${dir}") {
      sh """#!/bin/bash -e

cd ${workdir}

. /tools/local/bin/modinit.sh > /dev/null 2>&1
module use.own /proj/picasso/modulefiles

module add vivado/${version}_daily
module add sdaccel/${version}_daily
module add opencv/sdaccel

module add proxy
module add lftp
module add lsf

cd ${dir}

echo
echo "-----------------------------------------------"
echo "PWD: \$(pwd)"
echo "-----------------------------------------------"
echo

export PYTHONUNBUFFERED=true

export TMPDIR=\$(mktemp -d -p \$(pwd))

rm -rf \"out/${target}_${devdir}\" && mkdir -p out

bsub -W ${mins} -I -q ${queue} -R "osdistro=rhel && osver==ws6" -n ${cores} -R "rusage[mem=${mem}] span[ptile=${
        cores
      }]" -J "\$(basename ${dir})-${target}-run" <<EOF
#!/bin/bash -ex
export TMPDIR=\$TMPDIR
make TARGETS=${target} DEVICES=\"${device}\" NIMBIXFLAGS=\"--out out/${target}_${devdir} --queue_timeout=${mins}\" check
rm -rf \$TMPDIR
EOF

"""
//			}
    }
  }
}

def buildStatus(context, message, state) {
  step([$class            : 'GitHubCommitStatusSetter',
        contextSource     : [$class: 'ManuallyEnteredCommitContextSource', context: context],
        statusResultSource: [$class : 'ConditionalStatusResultSource',
                             results: [[$class: 'AnyBuildResult', message: message,
                                        state : state
                                       ]]
        ]
  ])

  return state
}

timestamps {
  node('xcoCentOS74Pool') {
    // hostname = sh(script: "hostname", returnStdout: true).trim()
    ws("/wrk/builds/sdaccel_examples/${env.BRANCH_NAME}") {
//      docker.image('sdaccel_examples').inside("-u xbuild -w /proj/xbb/sdaccel_examples --hostname ${hostname} -e VERSION=${VERSION} -v /proj/xbb:/proj/xbb -v /group/xcofarm:/group/xcofarm -v /tools/batonroot:/tools/batonroot -v /tools/dist:/tools/dist -v /proj/xbuilds:/proj/xbuilds") {
        try {
          stage("checkout") {
            checkout scm
          }

          precheck_status = buildStatus('ci-precheck', 'pre-build checks pending', 'PENDING')
          sw_emu_status = buildStatus('ci-sw_emu', 'sw_emu checks pending', 'PENDING')
          hw_status = buildStatus('ci-hw', 'hw checks pending', 'PENDING')

          stage("clean") {
            try {
              lastclean = readFile('lastclean.dat')
            } catch (e) {
              lastclean = "01/01/1970"
            } finally {
              lastcleanDate = new Date(lastclean)
              echo "Last Clean Build on ${lastcleanDate}"
            }

            def date = new Date()
            echo "Current Build Date is ${date}"

            if (date > lastcleanDate + days) {
              echo "Build too old resetting build area"
              sh 'git clean -xfd'
              // After clean write new lastclean Date to lastclean.dat
              writeFile(file: 'lastclean.dat', text: "${date}")
            }
          }

          stage('pre-check') {
            pre_check()
          }

          precheck_status = buildStatus('ci-precheck', 'pre-build checks passed', 'SUCCESS')

          workdir = pwd()

          def examples = []

          stage('configure') {
            sh 'utility/build_what.sh'
            examplesFile = readFile 'examples.dat'
            examples = examplesFile.split('\\n')

            def exSteps = [:]

            for (int i = 0; i < examples.size(); i++) {
              name = "${examples[i]}-setup"
              exSteps[name] = setupExample(examples[i], workdir)
            }

            parallel exSteps
          }

          def swBatches = devices.size() * 2
          def swEmuSteps = []
          def swEmuRunSteps = []

          for (int i = 0; i < swBatches; i++) {
            swEmuSteps[i] = [:]
            swEmuRunSteps[i] = [:]
          }

          for (int i = 0; i < examples.size(); i++) {
            for (int j = 0; j < devices.size(); j++) {
              batch = (j * examples.size() + i) % swBatches
              name = "${examples[i]}-${devices[j]}-sw_emu"
              swEmuSteps[batch]["${name}-build"] = buildExample('sw_emu', examples[i], devices[j], workdir)
              swEmuRunSteps[batch]["${name}-run"] = runExample('sw_emu', examples[i], devices[j], workdir)
            }
          }

          stage('sw_emu build') {
            for (int i = 0; i < swBatches; i++) {
              parallel swEmuSteps[i]
            }
          }

/* Disabled sw_emu while fixing a semaphore issue
    stage('sw_emu run') {
        lock("only_one_run_stage_at_a_time") {
            for(int i = 0; i < swBatches; i++) {
                parallel swEmuRunSteps[i]
            }
        }
    }
*/

        sw_emu_status = buildStatus('ci-sw_emu', 'sw_emu checks passed', 'SUCCESS')

        def hwBatches = devices.size() * 2
        def hwSteps = []
        def hwRunSteps = []

        for (int i = 0; i < hwBatches; i++) {
         hwSteps[i] = [:]
         hwRunSteps[i] = [:]
        }

        for (int i = 0; i < examples.size(); i++) {
         for (int j = 0; j < devices.size(); j++) {
            batch = (j * examples.size() + i) % hwBatches
            name = "${examples[i]}-${devices[j]}-hw"
            hwSteps[batch]["${name}-build"] = buildExample('hw', examples[i], devices[j], workdir)
            hwRunSteps[batch]["${name}-run"] = runExample('nimbix', examples[i], devices[j], workdir)
         }
        }

        stage('hw build') {
         for (int i = 0; i < hwBatches; i++) {
            try {
             parallel hwSteps[i]
            } catch (e) {
             hw_status = buildStatus('ci-hw', 'hw checks failed', 'FAILURE')
            }
         }
        }

        if (hw_status == "FAILURE") {
         throw RuntimeException("Failed to Build all Hardware Binaries");
        }
//
//        stage('hw run') {
//          lock("only_one_run_stage_at_a_time") {
//            for (int i = 0; i < hwBatches; i++) {
//              try {
//                parallel hwRunSteps[i]
//              } catch (e) {
//                hw_status = buildStatus('ci-hw', 'hw checks failed', 'FAILURE')
//              }
//            }
//          }
//        }
//
//        if (hw_status == "FAILURE") {
//          throw RuntimeException("Failed to all examples in hw");
//        }
//
//        hw_status = buildStatus('ci-hw', 'hw checks passed', 'SUCCESS')

        } catch (e) {
//        if (precheck_status == 'PENDING') {
//          precheck_status = buildStatus('ci-precheck', 'prechecks failed', 'FAILURE')
//        }
//        if (sw_emu_status == 'PENDING') {
//          sw_emu_status = buildStatus('ci-sw_emu', 'sw_emu checks failed', 'FAILURE')
//        }
//        if (hw_status == 'PENDING') {
//          hw_status = buildStatus('ci-hw', 'hw checks failed', 'FAILURE')
//        }

          currentBuild.result = "FAILED"
          throw e
        } finally {
//        stage('post-check') {
//          step([$class: 'Mailer', notifyEveryUnstableBuild: true, recipients: 'sdausr@xilinx.com', sendToIndividuals: false])
//        }
          stage('cleanup') {
            // Cleanup .Xil Files after run
            sh 'find . -name .Xil | xargs rm -rf'
          }
        } // try
//      } // docker
    } // ws
  } // node
} // timestamps
