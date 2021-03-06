// -*- mode: groovy -*-
// Jenkins pipeline
// See documents at https://jenkins.io/doc/book/pipeline/jenkinsfile/

// mxnet libraries
mx_lib = 'lib/libmxnet.so, lib/libmxnet.a, dmlc-core/libdmlc.a, nnvm/lib/libnnvm.a'
// command to start a docker container
docker_run = 'tests/ci_build/ci_build.sh'
// timeout in minutes
max_time = 60

// initialize source codes
def init_git() {
  retry(5) {
    try {
      timeout(time: 2, unit: 'MINUTES') {
        checkout scm
        sh 'git submodule update --init'
      }
    } catch (exc) {
      deleteDir()
      error "Failed to fetch source codes"
    }
  }
}

def init_git_win() {
  retry(5) {
    try {
      timeout(time: 2, unit: 'MINUTES') {
        checkout scm
        bat 'git submodule update --init'
      }
    } catch (exc) {
      deleteDir()
      error "Failed to fetch source codes"
    }
  }
}

stage("Sanity Check") {
  timeout(time: max_time, unit: 'MINUTES') {
    node('mxnetlinux') {
      ws('workspace/sanity') {
        init_git()
        make('lint', 'cpplint rcpplint jnilint')
        make('lint', 'pylint')
      }
    }
  }
}

// Run make. First try to do an incremental make from a previous workspace in hope to
// accelerate the compilation. If something wrong, clean the workspace and then
// build from scratch.
def make(docker_type, make_flag) {
  timeout(time: max_time, unit: 'MINUTES') {
    try {
      sh "newgrp docker"
      sh "${docker_run} ${docker_type} make ${make_flag}"
    } catch (exc) {
      echo 'Incremental compilation failed. Fall back to build from scratch'
      sh "newgrp docker"
      sh "${docker_run} ${docker_type} sudo make clean"
      sh "${docker_run} ${docker_type} make ${make_flag}"
    }
  }
}

// pack libraries for later use
def pack_lib(name, libs=mx_lib) {
  sh """
echo "Packing ${libs} into ${name}"
echo ${libs} | sed -e 's/,/ /g' | xargs md5sum
"""
  stash includes: libs, name: name
}


// unpack libraries saved before
def unpack_lib(name, libs=mx_lib) {
  unstash name
  sh """
echo "Unpacked ${libs} from ${name}"
echo ${libs} | sed -e 's/,/ /g' | xargs md5sum
"""
}

stage('Build') {
  parallel 'CPU: Openblas': {
    node('mxnetlinux') {
      ws('workspace/build-cpu') {
        init_git()
        def flag = """ \
DEV=1                         \
USE_PROFILER=1                \
USE_CPP_PACKAGE=1             \
USE_BLAS=openblas             \
-j\$(nproc)
"""
        make("cpu", flag)
        pack_lib('cpu')
      }
    }
  },
  'GPU: CUDA7.5+cuDNN5': {
    node('mxnetlinux') {
      ws('workspace/build-gpu') {
        init_git()
        def flag = """ \
DEV=1                         \
USE_PROFILER=1                \
USE_BLAS=openblas             \
USE_CUDA=1                    \
USE_CUDA_PATH=/usr/local/cuda \
USE_CUDNN=1                   \
USE_CPP_PACKAGE=1             \
-j\$(nproc)
"""
        make('gpu', flag)
        pack_lib('gpu')
        stash includes: 'build/cpp-package/example/test_score', name: 'cpp_test_score'
      }
    }
  },
  'Amalgamation': {
    node('mxnetlinux') {
      ws('workspace/amalgamation') {
        init_git()
        make('cpu', '-C amalgamation/ USE_BLAS=openblas MIN=1')
      }
    }
  },
  'GPU: MKLML': {
    node('mxnetlinux') {
      ws('workspace/build-mklml') {
        init_git()
        def flag = """ \
DEV=1                         \
USE_PROFILER=1                \
USE_BLAS=openblas             \
USE_MKL2017=1                 \
USE_MKL2017_EXPERIMENTAL=1    \
USE_CUDA=1                    \
USE_CUDA_PATH=/usr/local/cuda \
USE_CUDNN=1                   \
USE_CPP_PACKAGE=1             \
-j\$(nproc)
"""
        make('mklml_gpu', flag)
        pack_lib('mklml')
      }
    }
  }
}

// Python unittest for CPU
def python_ut(docker_type) {
  timeout(time: max_time, unit: 'MINUTES') {
    sh "newgrp docker"
    sh "${docker_run} ${docker_type} PYTHONPATH=./python/ nosetests --with-timer --verbose tests/python/unittest"
    sh "${docker_run} ${docker_type} PYTHONPATH=./python/ nosetests-3.4 --with-timer --verbose tests/python/unittest"
    sh "${docker_run} ${docker_type} PYTHONPATH=./python/ nosetests --with-timer --verbose tests/python/train"
  }
}

// GPU test has two parts. 1) run unittest on GPU, 2) compare the results on
// both CPU and GPU
def python_gpu_ut(docker_type) {
  timeout(time: max_time, unit: 'MINUTES') {
    sh "newgrp docker"
    sh "${docker_run} ${docker_type} PYTHONPATH=./python/ nosetests --with-timer --verbose tests/python/gpu"
    sh "${docker_run} ${docker_type} PYTHONPATH=./python/ nosetests-3.4 --with-timer --verbose tests/python/gpu"
  }
}

stage('Unit Test') {
  parallel 'Python2/3: CPU': {
    node('mxnetlinux') {
      ws('workspace/ut-python-cpu') {
        init_git()
        unpack_lib('cpu')
        python_ut('cpu')
      }
    }
  },
  'Python2/3: GPU': {
    node('mxnetlinux') {
      ws('workspace/ut-python-gpu') {
        init_git()
        unpack_lib('gpu', mx_lib)
        python_gpu_ut('gpu')
      }
    }
  },
  'Python2/3: MKLML': {
    node('mxnetlinux') {
      ws('workspace/ut-python-mklml') {
        init_git()
        unpack_lib('mklml')
        python_ut('mklml_gpu')
        python_gpu_ut('mklml_gpu')
      }
    }
  },
  'Scala: CPU': {
    node('mxnetlinux') {
      ws('workspace/ut-scala-cpu') {
        init_git()
        unpack_lib('cpu')
        timeout(time: max_time, unit: 'MINUTES') {
          sh "newgrp docker"
          sh "${docker_run} cpu make scalapkg USE_BLAS=openblas"
          sh "${docker_run} cpu make scalatest USE_BLAS=openblas"
        }
      }
    }
  },
  'R: CPU': {
    node('mxnetlinux') {
      ws('workspace/ut-r-cpu') {
        init_git()
        unpack_lib('cpu')
        timeout(time: max_time, unit: 'MINUTES') {
          sh "${docker_run} cpu rm -rf .Renviron"
          sh "${docker_run} cpu mkdir -p /workspace/ut-r-cpu/site-library"
          sh "${docker_run} cpu make rpkg USE_BLAS=openblas R_LIBS=/workspace/ut-r-cpu/site-library"
          sh "${docker_run} cpu R CMD INSTALL --library=/workspace/ut-r-cpu/site-library mxnet_current_r.tar.gz"
          sh "${docker_run} cpu make rpkgtest R_LIBS=/workspace/ut-r-cpu/site-library"
        }
      }
    }
  },
  'R: GPU': {
    node('mxnetlinux') {
      ws('workspace/ut-r-gpu') {
        init_git()
        unpack_lib('gpu')
        timeout(time: max_time, unit: 'MINUTES') {
          sh "${docker_run} cpu rm -rf .Renviron"
          sh "${docker_run} gpu mkdir -p /workspace/ut-r-gpu/site-library"
          sh "${docker_run} gpu make rpkg USE_BLAS=openblas R_LIBS=/workspace/ut-r-gpu/site-library"
          sh "${docker_run} gpu R CMD INSTALL --library=/workspace/ut-r-gpu/site-library mxnet_current_r.tar.gz"
          sh "${docker_run} gpu make rpkgtest R_LIBS=/workspace/ut-r-gpu/site-library"
        }
      }
    }
  },  
  'Python2/3: CPU Win':{
    node('mxnetwindows') {
      ws('workspace/ut-python-cpu') {
        init_git_win()
        unstash 'vc14_cpu'
        bat '''rmdir /s/q pkg_vc14_cpu
7z x -y vc14_cpu.7z'''
        bat """xcopy C:\\mxnet\\data data /E /I /Y
xcopy C:\\mxnet\\model model /E /I /Y
call activate py3
set PYTHONPATH=${env.WORKSPACE}\\pkg_vc14_cpu\\python
C:\\mxnet\\test_cpu.bat"""
                        bat """xcopy C:\\mxnet\\data data /E /I /Y
xcopy C:\\mxnet\\model model /E /I /Y
call activate py2
set PYTHONPATH=${env.WORKSPACE}\\pkg_vc14_cpu\\python
C:\\mxnet\\test_cpu.bat"""
      }
     }
   },
   'Python2/3: GPU Win':{
     node('mxnetwindows') {
       ws('workspace/ut-python-gpu') {
         init_git_win()
         unstash 'vc14_gpu'
         bat '''rmdir /s/q pkg_vc14_gpu
7z x -y vc14_gpu.7z'''
         bat """xcopy C:\\mxnet\\data data /E /I /Y
xcopy C:\\mxnet\\model model /E /I /Y
call activate py3
set PYTHONPATH=${env.WORKSPACE}\\pkg_vc14_gpu\\python
C:\\mxnet\\test_gpu.bat"""
         bat """xcopy C:\\mxnet\\data data /E /I /Y
xcopy C:\\mxnet\\model model /E /I /Y
call activate py2
set PYTHONPATH=${env.WORKSPACE}\\pkg_vc14_gpu\\python
C:\\mxnet\\test_gpu.bat"""
       }
     }
   }
}


stage('Integration Test') {
  parallel 'Python': {
    node('mxnetlinux') {
      ws('workspace/it-python-gpu') {
        init_git()
        unpack_lib('gpu')
        timeout(time: max_time, unit: 'MINUTES') {
          sh "newgrp docker"
          sh "${docker_run} gpu PYTHONPATH=./python/ python example/image-classification/test_score.py"
        }
      }
    }
  },
  'Caffe': {
    node('mxnetlinux') {
      ws('workspace/it-caffe') {
        init_git()
        unpack_lib('gpu')
        timeout(time: max_time, unit: 'MINUTES') {
          sh "newgrp docker"
          sh "${docker_run} caffe_gpu PYTHONPATH=/caffe/python:./python python tools/caffe_converter/test_converter.py"
        }
      }
    }
  },
  'cpp-package': {
    node('mxnetlinux') {
      ws('workspace/it-cpp-package') {
        init_git()
        unpack_lib('gpu')
        unstash 'cpp_test_score'
        timeout(time: max_time, unit: 'MINUTES') {
          sh "newgrp docker"
          sh "${docker_run} gpu cpp-package/tests/ci_test.sh"
        }
      }
    }
  }
}

stage('Deploy') {
  node('mxnetlinux') {
    ws('workspace/docs') {
      if (env.BRANCH_NAME == "master") {
        init_git()
        sh "make docs"
      }
    }
  }
}
