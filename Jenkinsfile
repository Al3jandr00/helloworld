pipeline {
   
   agent any
   
   /*
           Get Code
   */
   
    stages {
        stage('Get Code') {
            steps {
                git 'https://github.com/Al3jandr00/helloworld.git'
            }
        }
        
        /* 
           Setup virtualenv
         */

        stage('Setup venv') {
            steps {
                bat '''
C:\\Users\\AlejandroFraga\\AppData\\Local\\Programs\\Python\\Python314\\python.exe -m venv venv
venv\\Scripts\\pip install --upgrade pip
venv\\Scripts\\pip install pytest flake8 bandit coverage
'''
            }
        }
        /* 
           Unit Tests (pytest + coverage)
           UNA sola vez
         */

stage('Unit') {
    steps {
        bat """
        set PYTHONPATH=%WORKSPACE%
        venv\\Scripts\\coverage run --branch -m pytest --junitxml=result-unit.xml test\\unit
        venv\\Scripts\\coverage xml
        """
    }
    post {
        always {
            junit 'result-unit.xml'
        }
    }
}

        /* 
           Coverage Quality Gates
         */

     stage('Coverage') {
            steps {
                recordCoverage(
                    tools: [[parser: 'COBERTURA', pattern: 'coverage.xml']],
                    qualityGates: [
                        // LINE coverage
                        [metric: 'LINE', threshold: 95.0, criticality: 'NOTE'],
                        [metric: 'LINE', threshold: 85.0, criticality: 'ERROR'],

                        // BRANCH coverage
                        [metric: 'BRANCH', threshold: 90.0, criticality: 'NOTE'],
                        [metric: 'BRANCH', threshold: 80.0, criticality: 'ERROR']
                    ]
                )
            }
        }
        
         /* 
           Service (REST tests)
           Flask + WireMock
         */

stage('Service') {
    steps {
        bat '''
        set PYTHONPATH=%WORKSPACE%
        set FLASK_APP=app\\api.py
        
        start /B java -jar tools\\wiremock\\wiremock-jre8-standalone-2.28.0.jar ^
        --port 9090 ^
        -v ^
        --root-dir test\\wiremock > wiremock.log 2>&1

        REM  Arrancar Flask 
        start /B flask run > flask.log 2>&1

        REM Pausa para que arranquen FLASK y WIREMOCK
        ping 127.0.0.1 -n 5 > nul

        REM Tests REST
        venv\\Scripts\\pytest --junitxml=result-rest.xml test\\rest || echo REST tests failed

        REM Parar Flask (puerto 5000) para volver a lanzarlo en el stage Performance
       for /f "tokens=5" %%a in ('netstat -ano ^| findstr LISTENING ^| findstr :5000') do (
       taskkill /PID %%a /F
       )
        '''
    }
    post {
        always {
            junit 'result-rest.xml'
        }
    }
}

        /* 
           Static Analysis (Flake8)
         */

stage('Static Analysis') {
    steps {
        script {
            
            bat """
                venv\\Scripts\\flake8 app --count --statistics > flake8-report.txt 2>&1
                exit /B 0
            """
        }
    }

    post {
        always {
            recordIssues(
                tools: [flake8(pattern: 'flake8-report.txt')],
                enabledForFailure: true,

                // QUALITY GATES
                qualityGates: [
                    [threshold: 8, type: 'TOTAL', unstable: true],
                    [threshold: 10, type: 'TOTAL', unstable: false]
                ]
            )
        }
    }
}

        /* 
           Security Analysis (Bandit - Limitamos los errores a: B310)
         */

stage('Security Analysis') {
    steps {
        bat '''
        venv\\Scripts\\bandit ^
          --exit-zero ^
          -r . ^
          -t B310 ^
          -f custom ^
          -o bandit.out ^
          --msg-template "{abspath}:{line}: [{test_id}] {msg}"
        '''
        recordIssues(
            tools: [pyLint(name: 'Bandit', pattern: 'bandit.out')],
            qualityGates: [
                [threshold: 2, type: 'TOTAL', unstable: true],
                [threshold: 4, type: 'TOTAL', unstable: false]
            ]
        )
    }
}

        /* 
           Performance (JMeter)
         */
         
         
            stage('Performance') {
            steps {
                bat '''
                set FLASK_APP=app\\api.py

                REM --- Start Flask ---
                start /B flask run > flask_perf.log 2>&1
                ping 127.0.0.1 -n 5 > nul

                REM --- JMeter ---
                C:\\Users\\AlejandroFraga\\Desktop\\CURSOS\\UNIR_DEVOPS\\apache-jmeter-5.6.3\\bin\\jmeter ^
                  -n ^
                  -t test\\jmeter\\Test_Plan.jmx ^
                  -f ^
                  -l Test_Plan.jtl

                REM --- Stop Flask ---
                for /f "tokens=5" %%a in ('netstat -ano ^| findstr LISTENING ^| findstr :5000') do (
                    taskkill /PID %%a /F
                )
                '''
            }
            post {
                always {
                    perfReport sourceDataFiles: 'Test_Plan.jtl'
                }
            }
        }

}
}