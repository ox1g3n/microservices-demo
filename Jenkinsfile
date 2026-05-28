pipeline {
    agent any

    environment {
        GEMINI_API_KEY       = credentials('gemini-api-key')
        COMPOSE_PROJECT_NAME = 'sockshop-chaos-demo'
        COMPOSE_FILE         = 'docker-compose.yml'
        APP_NETWORK          = 'sockshop-chaos-demo_default'
        CHAOS_IMAGE          = 'chaos-controller:latest'
        CHAOS_CONTAINER      = 'chaos-controller-sockshop'
    }

    stages {

        stage('Build & Start Sock Shop') {
            steps {
                echo '🔨 Building and starting the Sock Shop stack (13 microservices)...'
                sh "docker-compose -f ${COMPOSE_FILE} up --build -d"
                echo '⏳ Waiting 60s for all services to initialize...'
                sh 'sleep 60'
                echo '✅ Sock Shop stack is up.'
            }
        }

        stage('Start ChaosController') {
            steps {
                echo '🔥 Starting ChaosController...'
                sh "docker rm -f ${CHAOS_CONTAINER} 2>/dev/null || true"
                sh """
                    docker run -d \
                        --name ${CHAOS_CONTAINER} \
                        --network ${APP_NETWORK} \
                        -p 5050:5050 \
                        -v /var/run/docker.sock:/var/run/docker.sock \
                        -e GEMINI_API_KEY=${GEMINI_API_KEY} \
                        ${CHAOS_IMAGE}
                """
                sh "docker network connect ${APP_NETWORK} jenkins || true"
                sh '''
                    for i in $(seq 1 30); do
                        curl -sf http://chaos-controller-sockshop:5050/status > /dev/null && echo "ChaosController ready!" && exit 0
                        echo "Waiting for ChaosController... ($i/30)"
                        sleep 5
                    done
                    echo "ChaosController failed to start!" && exit 1
                '''
                echo '✅ ChaosController is ready.'
            }
        }

        stage('Application Health Check') {
            steps {
                echo '🏥 Verifying Sock Shop services are healthy...'
                sh '''
                    for i in $(seq 1 20); do
                        curl -sf http://edge-router:80/ > /dev/null && echo "Sock Shop healthy!" && exit 0
                        echo "Waiting for Sock Shop... ($i/20)"
                        sleep 5
                    done
                    echo "Sock Shop failed to become healthy!" && exit 1
                '''
                echo '✅ Sock Shop is healthy and ready for chaos testing.'
            }
        }

        stage('Resilience Testing Gate') {
            steps {
                script {
                    def agentHost = sh(script: "hostname -I | awk '{print \$1}'", returnStdout: true).trim()

                    echo """
╔══════════════════════════════════════════════════════════════╗
║           🔥  RESILIENCE TESTING GATE  🔥                    ║
╠══════════════════════════════════════════════════════════════╣
║                                                              ║
║   Dashboard: http://${agentHost}:5050                        ║
║   Sock Shop: http://${agentHost}:8083                        ║
║                                                              ║
║   13 microservices have been auto-discovered.                ║
║   Open the dashboard to:                                     ║
║     • Inject faults (latency, crash, CPU, packet loss...)   ║
║     • Run chaos experiments on front-end, carts, orders     ║
║     • Get AI-powered remediation reports (Gemini)            ║
║     • Test user authentication under chaos conditions       ║
║                                                              ║
║   When satisfied, come back here and click APPROVE.          ║
║                                                              ║
╚══════════════════════════════════════════════════════════════╝
                    """

                    input(
                        id: 'ResilienceApproval',
                        message: 'Resilience Testing Complete?',
                        ok: 'Approve — System is resilient, continue to deploy'
                    )

                    echo '✅ Resilience gate APPROVED.'
                }
            }
        }

        stage('Deploy to Production') {
            steps {
                echo '🚀 Resilience gate passed — deploying to production...'
                echo '(In a real scenario, this would run: kubectl apply, helm upgrade, etc.)'
                echo '✅ Deployed successfully!'
            }
        }
    }

    post {
        always {
            echo '🧹 Cleaning up...'
            sh "docker stop ${CHAOS_CONTAINER} 2>/dev/null || true"
            sh "docker rm ${CHAOS_CONTAINER} 2>/dev/null || true"
            sh "docker-compose -f ${COMPOSE_FILE} down 2>/dev/null || true"
        }
        failure {
            echo '❌ Pipeline failed or was rejected.'
        }
        success {
            echo '✅ Pipeline succeeded!'
        }
    }
}