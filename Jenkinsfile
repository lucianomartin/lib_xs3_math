@Library('xmos_jenkins_shared_library@v0.27.0')

def runningOn(machine) {
  println "Stage running on:"
  println machine
}

getApproval()
pipeline {
  agent none

  parameters {
    string(
      name: 'TOOLS_VERSION',
      defaultValue: '15.2.1',
      description: 'The XTC tools version'
    )
    booleanParam(
      name: 'XMATH_SMOKE_TEST',
      defaultValue: true,
      description: 'Enable smoke run'
    )
  } // parameters
  options {
    skipDefaultCheckout()
    timestamps()
    buildDiscarder(xmosDiscardBuildSettings())
  } // options

  stages {
    stage('Bullds and tests') {
      parallel {
        stage('Linux builds and tests') {
          agent {
            label 'xcore.ai'
          }
          stages {
            stage('Build') {
              steps {
                runningOn(env.NODE_NAME)
                dir('lib_xcore_math') {
                  checkout scm
                  // fetch submodules
                  sh 'git submodule update --init --recursive --jobs 4'
                  withTools(params.TOOLS_VERSION) {
                    // xs3a build
                    sh "cmake -B build_xs3a -DXMATH_SMOKE_TEST=${params.XMATH_SMOKE_TEST} --toolchain=etc/xmos_cmake_toolchain/xs3a.cmake"
                    sh 'make -C build_xs3a -j4'
                    // x86 build
                    sh "cmake -B build_x86 -DXMATH_SMOKE_TEST=${params.XMATH_SMOKE_TEST}"
                    sh 'make -C build_x86 -j4'
                    // xmake build
                    dir('test/legacy_build') {
                      sh 'xmake -j4'
                      sh 'xrun --io --id 0 bin/legacy_build.xe'
                    }
                  }
                }
              }
            } // Build

            stage('Unit tests xs3a') {
              steps {
                dir('lib_xcore_math/build_xs3a/test') {
                  withTools(params.TOOLS_VERSION) {
                    sh 'xrun --xscope --id 0 --args bfp_tests/bfp_tests.xe        -v'
                    sh 'xrun --xscope --id 0 --args dct_tests/dct_tests.xe        -v'
                    sh 'xrun --xscope --id 0 --args fft_tests/fft_tests.xe        -v'
                    sh 'xrun --xscope --id 0 --args filter_tests/filter_tests.xe  -v'
                    sh 'xrun --xscope --id 0 --args scalar_tests/scalar_tests.xe  -v'
                    sh 'xrun --xscope --id 0 --args vect_tests/vect_tests.xe      -v'
                    sh 'xrun --xscope --id 0 --args xs3_tests/xs3_tests.xe        -v'
                  }
                }
              }
            } // Unit tests xs3a

            stage('Unit tests x86') {
              steps {
                dir('lib_xcore_math/build_x86/test') {
                  sh './bfp_tests/bfp_tests        -v'
                  sh './dct_tests/dct_tests        -v'
                  sh './fft_tests/fft_tests        -v'
                  sh './filter_tests/filter_tests  -v'
                  sh './scalar_tests/scalar_tests  -v'
                  sh './vect_tests/vect_tests      -v'
                  sh './xs3_tests/xs3_tests        -v'
                }
              }
            } // Unit tests x86
          } // stages
          post {
            cleanup {
              cleanWs()
            }
          }
        } // Linux builds and tests

        stage('Windows builds and tests') {
          agent {
            label 'windows10&&unified'
          }
          stages {
            stage('Build') {
              steps {
                runningOn(env.NODE_NAME)
                dir('lib_xcore_math') {
                  checkout scm
                  // fetch submodules
                  bat 'git submodule update --init --recursive --jobs 4'
                  withTools(params.TOOLS_VERSION) {
                    withVS {
                      // xs3a build
                      bat 'cmake -B build_xs3a -DXMATH_SMOKE_TEST=${params.XMATH_SMOKE_TEST} --toolchain=etc/xmos_cmake_toolchain/xs3a.cmake -G"Ninja"'
                      bat 'ninja -C build_xs3a -j4'
                      // x86 build
                      bat 'cmake -B build_x86 -DXMATH_SMOKE_TEST=${params.XMATH_SMOKE_TEST} -G"Ninja"'
                      bat 'ninja -C build_x86 -j4'
                      // xmake build
                      dir('test/legacy_build') {
                        bat 'xmake --jobs 4'
                      }
                    }
                  }
                }
              }
            } // Build

            stage('Unit tests x86') {
              steps {
                dir('lib_xcore_math/build_x86/test') {
                  bat './bfp_tests/bfp_tests.exe        -v'
                  bat './dct_tests/dct_tests.exe        -v'
                  bat './fft_tests/fft_tests.exe        -v'
                  bat './filter_tests/filter_tests.exe  -v'
                  bat './scalar_tests/scalar_tests.exe  -v'
                  bat './vect_tests/vect_tests.exe      -v'
                  bat './xs3_tests/xs3_tests.exe        -v'
                }
              }
            } // Unit tests x86
          } // stages
          post {
            cleanup {
              xcoreCleanSandbox()
            }
          }
        } // Windows builds and tests

        stage ('Build Documentation') {
          agent {
            label 'docker'
          }
          stages {
            stage('Build Docs') {
              steps {
                runningOn(env.NODE_NAME)
                checkout scm
                sh """docker run --user "\$(id -u):\$(id -g)" \
                        --rm \
                        -v ${WORKSPACE}:/build \
                        -e EXCLUDE_PATTERNS="/build/doc/doc_excludes.txt" \
                        -e DOXYGEN=1 -e DOXYGEN_INCLUDE=/build/doc/Doxyfile.inc \
                        -e PDF=1 \
                        ghcr.io/xmos/doc_builder:v3.0.0 \
                        || echo "PDF build is badly broken, ignoring for now till it's fixed." """

                archiveArtifacts artifacts: "doc/_build/**", allowEmptyArchive: true
              } // steps
            } // Build Docs
          } // stages
          post {
            cleanup {
              cleanWs()
            }
          }
        } // Build Documentation

      } // parallel
    } // Bullds and tests
  } // stages
} // pipeline
