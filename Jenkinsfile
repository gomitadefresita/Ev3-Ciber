// Jenkinsfile - Pipeline de "Sistema Alumnos" (SecureDev · Evaluación Parcial 3 · OCY1102)
//
// Incluye Build, Test, revisiones de seguridad (SAST y dependencias), staging, DAST y despliegue en producción.
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
    }

    stages {
        stage('Build') {
            steps {
                script {
                    // Versión = 1.0.<nº de build>+<commit corto>: une cada despliegue con su commit (trazabilidad)
                    def sha = sh(returnStdout: true, script: 'git rev-parse --short HEAD').trim()
                    env.APP_VERSION = "1.0.${env.BUILD_NUMBER}+${sha}"
                }
                echo "Versión de este build: ${env.APP_VERSION}"
                sh 'git log -1 --format="Commit %h de %an: %s"'
                // Cada build parte limpio: sin reportes viejos y con un entorno virtual nuevo
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
                sh '.venv/bin/bandit -r app.py -f json -o reports/bandit.json || true'
                sh '.venv/bin/bandit -r app.py || true'
            }
        }

        stage('Dependency Management') {
            steps {
                sh '.venv/bin/pip-audit -f json -o reports/pip-audit.json || true'
                sh '.venv/bin/pip-audit'
            }
        }

        stage('Deploy Staging') {
            steps {
                sh 'bash scripts/deploy.sh staging 5001 || bash scripts/deploy.sh staging 5000'
            }
        }

        stage('DAST - OWASP ZAP') {
            steps {
                sh '''
                    mkdir -p reports
                    TARGET_URL="http://127.0.0.1:5001"
                    if ! curl -sf "$TARGET_URL/health" > /dev/null 2>&1; then
                        TARGET_URL="http://127.0.0.1:5000"
                    fi

                    if command -v zap-baseline.py >/dev/null 2>&1; then
                        zap-baseline.py -t "$TARGET_URL" -r zap-report.html -J zap-report.json || true
                        cp /zap/wrk/zap-report.* reports/ 2>/dev/null || true
                    elif command -v docker >/dev/null 2>&1; then
                        docker run --rm --network host -v "$(pwd)/reports:/zap/wrk/:rw" \
                            ghcr.io/zaproxy/zaproxy:stable zap-baseline.py \
                            -t "$TARGET_URL" -r zap-report.html -J zap-report.json || true
                    else
                        echo "ZAP no está instalado en este entorno; revisa el agente o contenedor de Jenkins."
                    fi
                '''
            }
        }

        stage('Deploy Producción') {
            // >>> [GATE]: aquí va la condición "when" que protege producción
            steps {
                sh 'bash scripts/deploy.sh produccion 5000'
                script { env.PRODUCCION = env.APP_VERSION }
            }
        }
    }

    post {
        always {
            // Registro de trazabilidad: build <-> versión <-> commit <-> pruebas <-> hallazgos <-> despliegue
            withEnv(["RESULTADO=${currentBuild.currentResult}"]) {
                sh 'python3 scripts/trazabilidad.py'
            }
            archiveArtifacts artifacts: 'reports/**', allowEmptyArchive: true
        }
    }
}