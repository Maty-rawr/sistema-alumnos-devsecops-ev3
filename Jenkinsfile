// Jenkinsfile - Pipeline de "Sistema Alumnos" (SecureDev · Evaluación Parcial 3 · OCY1102)
pipeline {
    agent any

    options {
        timestamps()
        timeout(time: 40, unit: 'MINUTES')
        disableConcurrentBuilds()
    }

    // Revisa GitHub cada ~3 minutos: cada push (tuyo o un PR de Dependabot fusionado) dispara un build.
    triggers {
        pollSCM('H/3 * * * *')
    }

    environment {
        PIP_DISABLE_PIP_VERSION_CHECK = '1'
        // En la red interna de Docker, el servicio de staging se alcanza por el nombre del contenedor/servicio.
        // Si en tu compose.yaml el servicio de staging tiene otro nombre, ajústalo aquí (ej. http://taller:5001).
        ZAP_TARGET_URL = 'http://taller:5001'
    }

    stages {
        stage('Build') {
            steps {
                script {
                    def sha = sh(returnStdout: true, script: 'git rev-parse --short HEAD').trim()
                    env.APP_VERSION = "1.0.${env.BUILD_NUMBER}+${sha}"
                }
                echo "Versión de este build: ${env.APP_VERSION}"
                sh 'git log -1 --format="Commit %h de %an: %s"'
                sh 'rm -rf .venv reports && mkdir -p reports'
                sh 'python3 -m venv .venv'
                sh '.venv/bin/pip install --quiet -r requirements.txt -r requirements-dev.txt'
                sh '.venv/bin/python -m compileall -q app.py'
            }
        }

        stage('Test') {
            steps {
                sh '.venv/bin/python -m pytest --junitxml=reports/junit.xml'
            }
            post {
                always {
                    junit allowEmptyResults: true, testResults: 'reports/junit.xml'
                }
            }
        }

        stage('Security Review (SAST)') {
            steps {
                sh '''
                    cd "${WORKSPACE}"
                    .venv/bin/bandit -r . -x ./.venv,./tests -ll -f txt -o reports/bandit.txt || true
                    cat reports/bandit.txt || true
                '''
            }
        }

        stage('Dependency Management') {
            steps {
                sh '''
                    cd "${WORKSPACE}"
                    .venv/bin/pip-audit -r requirements.txt -f json -o reports/pip-audit.json || true
                    .venv/bin/pip-audit -r requirements.txt || true
                '''
            }
        }

        stage('Deploy Staging') {
            steps {
                sh '''
                    cd "${WORKSPACE}"
                    bash scripts/deploy.sh staging 5001
                '''
            }
        }

        stage('DAST - OWASP ZAP') {
            steps {
                sh '''
                    cd "${WORKSPACE}"
                    mkdir -p reports
                    OBJETIVO="${ZAP_TARGET_URL:-http://127.0.0.1:5001}"
                    echo "Ejecutando escaneo OWASP ZAP contra: ${OBJETIVO}"
                    set +e
                    .venv/bin/python scripts/zap_scan.py "${OBJETIVO}"
                    RC=$?
                    set -e
                    if [ "$RC" -eq 2 ]; then
                        echo "AVISO: OWASP ZAP detectó alertas High/Medium (código 2); se conserva el reporte en reports/."
                    elif [ "$RC" -ne 0 ]; then
                        echo "ERROR: zap_scan.py falló con código ${RC}"
                        exit "$RC"
                    fi
                '''
            }
            post {
                always {
                    script {
                        if (fileExists('reports/zap-resumen.txt')) {
                            echo readFile('reports/zap-resumen.txt')
                        }
                    }
                }
            }
        }

        stage('Deploy Producción') {
            when {
                allOf {
                    branch 'main'
                    expression { currentBuild.currentResult != 'FAILURE' }
                }
            }
            steps {
                sh '''
                    cd "${WORKSPACE}"
                    bash scripts/deploy.sh produccion 5000
                '''
                script {
                    env.PRODUCCION = env.APP_VERSION
                }
            }
        }
    }

    post {
        always {
            sh '''
                cd "${WORKSPACE}"
                mkdir -p reports
            '''
            withEnv(["RESULTADO=${currentBuild.currentResult}"]) {
                sh '''
                    cd "${WORKSPACE}"
                    python3 scripts/trazabilidad.py || true
                '''
            }
            archiveArtifacts artifacts: 'reports/**', allowEmptyArchive: true
        }
    }
}