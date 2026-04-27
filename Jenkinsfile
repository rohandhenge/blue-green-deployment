pipeline {
    agent any

    environment {

        //  CHANGE HERE → Blue Target Group ARN
        BLUE_TG  = 'arn:aws:elasticloadbalancing:ap-northeast-2:427617722045:targetgroup/TG-Blue/51d1c11587f20f1f'

        //  CHANGE HERE → Green Target Group ARN
        GREEN_TG = 'arn:aws:elasticloadbalancing:ap-northeast-2:427617722045:targetgroup/TG-Green/e3c645fea483d2e3'

        //  CHANGE HERE → ALB Listener ARN
        LISTENER_ARN = 'arn:aws:elasticloadbalancing:ap-northeast-2:427617722045:listener/app/blue-green-alb/77968cc89beecbee/70ea8976420306be'

        //  CHANGE HERE → Green server public IP
        GREEN_IP = '43.203.224.107'
    }

    stages {

        stage('Deploy to Green') {
            steps {
                sh """
                echo "Deploying to GREEN server..."

                # Copy file to server
                scp -o StrictHostKeyChecking=no index.html ubuntu@${GREEN_IP}:/tmp/

                # Move file to nginx directory
                ssh -o StrictHostKeyChecking=no ubuntu@${GREEN_IP} sudo mv /tmp/index.html /var/www/html/index.html
                """
            }
        }

        stage('Health Check') {
            steps {
                script {
                    echo "Checking application health..."

                    sleep 15

                    def status = sh(
                        script: "curl -s -o /dev/null -w '%{http_code}' http://${GREEN_IP}",
                        returnStdout: true
                    ).trim()

                    echo "HTTP Status: ${status}"

                    if (status != "200") {
                        error("❌ Health check failed")
                    } else {
                        echo "✅ Health check passed"
                    }
                }
            }
        }

        stage('Switch Traffic') {
            steps {
                sh """
                echo "Switching traffic to GREEN..."

                aws elbv2 modify-listener \
                --listener-arn ${LISTENER_ARN} \
                --default-actions Type=forward,TargetGroupArn=${GREEN_TG}
                """
            }
        }
    }

    post {
        failure {
            echo "❌ Deployment failed — rolling back to BLUE..."

            sh """
            aws elbv2 modify-listener \
            --listener-arn ${LISTENER_ARN} \
            --default-actions Type=forward,TargetGroupArn=${BLUE_TG}
            """
        }
    }
}
