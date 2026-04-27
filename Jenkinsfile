pipeline {
    agent any

    environment {
        LISTENER_ARN = 'arn:aws:elasticloadbalancing:ap-northeast-2:427617722045:listener/app/blue-green-alb/77968cc89beecbee/70ea8976420306be'  // ← CHANGE
        BLUE_TG_ARN   = 'arn:aws:elasticloadbalancing:ap-northeast-2:427617722045:targetgroup/TG-Blue/51d1c11587f20f1f'       // ← CHANGE
        GREEN_TG_ARN  = 'arn:aws:elasticloadbalancing:ap-northeast-2:427617722045:targetgroup/TG-Green/e3c645fea483d2e3'    // ← CHANGE
        BLUE_IP       = '13.125.200.20'   // Blue instance public IP
        GREEN_IP      = '43.203.224.107'   // Green instance public IP
    }

    stages {

        stage('Deploy to Green') {
            steps {
                sh """
                scp -o StrictHostKeyChecking=no index.html ubuntu@${GREEN_IP}:/tmp/
                ssh -o StrictHostKeyChecking=no ubuntu@${GREEN_IP} "sudo mv /tmp/index.html /var/www/html/index.html"
                """
            }
        }

        stage('Health Check') {
            steps {
                script {
                    sleep 15
                    def status = sh(
                        script: "curl -s -o /dev/null -w '%{http_code}' http://${GREEN_IP}",
                        returnStdout: true
                    ).trim()

                    if (status != "200") {
                        error("Health check failed")
                    }
                }
            }
        }

        stage('Switch Traffic') {
            steps {
                sh """
                aws elbv2 modify-listener \
                --listener-arn ${LISTENER_ARN} \
                --default-actions Type=forward,TargetGroupArn=${GREEN_TG}
                """
            }
        }
    }

    post {
        failure {
            echo "Rollback triggered"
            sh """
            aws elbv2 modify-listener \
            --listener-arn ${LISTENER_ARN} \
            --default-actions Type=forward,TargetGroupArn=${BLUE_TG}
            """
        }
    }
}
