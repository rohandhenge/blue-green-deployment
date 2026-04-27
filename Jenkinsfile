pipeline {
    agent any
    parameters {
        choice(name: 'DEPLOY_TO', choices: ['blue', 'green'], description: 'Which environment is INACTIVE right now?')
        booleanParam(name: 'AUTO_SWITCH', defaultValue: true, description: 'Switch traffic after success?')
    }

    environment {
        LISTENER_ARN = 'arn:aws:elasticloadbalancing:ap-northeast-2:427617722045:listener/app/blue-green-alb/77968cc89beecbee/70ea8976420306be'  // ← CHANGE
        BLUE_TG_ARN   = 'arn:aws:elasticloadbalancing:ap-northeast-2:427617722045:targetgroup/TG-Blue/51d1c11587f20f1f'       // ← CHANGE
        GREEN_TG_ARN  = 'arn:aws:elasticloadbalancing:ap-northeast-2:427617722045:targetgroup/TG-Green/e3c645fea483d2e3'    // ← CHANGE
        BLUE_IP       = '13.125.200.20'   // Blue instance public IP
        GREEN_IP      = '43.203.224.107'   // Green instance public IP
    }

    stages {
        stage('Checkout Code') {
            steps {
                git branch: 'main', url: 'https://github.com/rohandhenge/blue-green-deployment'
            }
        }

        stage('Determine Inactive Environment') {
            steps {
                script {
                    if (params.DEPLOY_TO == 'blue') {
                        env.INACTIVE_IP = env.BLUE_IP
                        env.INACTIVE_TG = env.BLUE_TG_ARN
                        env.ACTIVE_TG   = env.GREEN_TG_ARN
                    } else {
                        env.INACTIVE_IP = env.GREEN_IP
                        env.INACTIVE_TG = env.GREEN_TG_ARN
                        env.ACTIVE_TG   = env.BLUE_TG_ARN
                    }
                    echo "Deploying to inactive: ${params.DEPLOY_TO}"
                }
            }
        }

        stage('Deploy New Version to Inactive Env') {
            steps {
                sshagent(credentials: ['ec2-ssh-key']) {
                    sh """
                        ssh -o StrictHostKeyChecking=no ubuntu@${INACTIVE_IP} '
                            sudo rm -rf /var/www/html/*
                            sudo mkdir -p /var/www/html
                        '
                        scp -r * ubuntu@${INACTIVE_IP}:/var/www/html/
                        ssh ubuntu@${INACTIVE_IP} '
                            sudo systemctl restart nginx
                            echo "Deployed Version \$(cat index.html | grep Version)" 
                        '
                    """
                }
            }
        }

        stage('Health Check') {
            steps {
                script {
                    echo "Waiting for health check..."
                    sleep 10
                    def health = sh(script: "aws elbv2 describe-target-health --target-group-arn ${INACTIVE_TG} --query 'TargetHealthDescriptions[0].TargetHealth.State' --output text", returnStdout: true).trim()
                    if (health != "healthy") {
                        error "Health check FAILED on ${params.DEPLOY_TO}"
                    }
                    echo "✅ Health check PASSED"
                }
            }
        }

        stage('Switch Traffic') {
            when { expression { return params.AUTO_SWITCH } }
            steps {
                sh """
                    aws elbv2 modify-listener \
                    --listener-arn ${LISTENER_ARN} \
                    --default-actions Type=forward,TargetGroupArn=${INACTIVE_TG}
                """
                echo "🚀 TRAFFIC SWITCHED to ${params.DEPLOY_TO}!"
            }
        }
    }

    post {
        failure {
            echo "❌ Deployment failed - Rolling back"
            sh """
                aws elbv2 modify-listener \
                --listener-arn ${LISTENER_ARN} \
                --default-actions Type=forward,TargetGroupArn=${ACTIVE_TG}
            """
        }
        always {
            echo "Pipeline finished. Check ALB DNS to verify."
        }
    }
}
