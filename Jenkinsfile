pipeline {
    agent any

    environment {
        EC2_USER = 'ubuntu'
        EC2_HOST = '35.159.66.99' // EC2-IP anpassen
        DEPLOY_PATH = '/var/www/nginx/react-app' // Nginx-Standardverzeichnis
        SSH_CREDENTIALS = 'jenkins-ec2-key'  
        APP_URL = 'http://35.159.66.99' 
    }
    
    tools {
        nodejs 'NodeJS'
    }

    stages {
        stage('Clone Repository') {
            steps {
                git branch: 'main', 
                    url: 'https://github.com/marcusBieber/react-build.git' // add some other git repository for deployment
            }
        }

        stage('Install Dependencies and Build React App') {
            steps {
                sh '''
                    npm install
                    npm run build
                '''
            }
        }

        stage('Deploy to EC2') {
            steps {
                script {
                    sshagent(credentials: [SSH_CREDENTIALS]) {
                        sh """
                        ssh -o StrictHostKeyChecking=no ${EC2_USER}@${EC2_HOST} '
                            echo "Prüfe Nginx-Installation..." &&
                            if ! command -v nginx &> /dev/null; then
                                echo "Nginx wird installiert..." &&
                                sudo apt update &&
                                sudo apt install -y nginx &&
                                sudo systemctl enable nginx &&
                                sudo systemctl start nginx
                            fi &&
                            
                            echo "Erstelle Deployment-Verzeichnis..." &&
                            sudo mkdir -p ${DEPLOY_PATH} &&
                            sudo chown -R ubuntu:ubuntu ${DEPLOY_PATH} &&
                            sudo rm -rf ${DEPLOY_PATH}/* &&
                            echo "Entferne Standard-Nginx-Site..." &&
                            sudo rm -f /etc/nginx/sites-enabled/default
                            '
                        
                        echo "Kopiere neuen Build auf EC2..."
                        scp -o StrictHostKeyChecking=no -r dist/* ${EC2_USER}@${EC2_HOST}:${DEPLOY_PATH}

                        ssh -o StrictHostKeyChecking=no ${EC2_USER}@${EC2_HOST} '
                            sudo tee /etc/nginx/sites-available/react-app > /dev/null <<EOT
server {
    listen 80;
    server_name _;
    root ${DEPLOY_PATH};
    index index.html;
    
    location / {
        try_files \\\$uri /index.html;
    }

    error_page 404 /index.html;
}
EOT
                            sudo ln -sf /etc/nginx/sites-available/react-app /etc/nginx/sites-enabled/react-app &&
                            sudo systemctl restart nginx'
                        """
                    }
                }
            }
        }

        stage('Check Website Availability') {
            steps {
                script {
                    def response = sh(script: "curl -o /dev/null -s -w \"%{http_code}\" ${APP_URL}", returnStdout: true).trim()
                    if (response != "200") {
                        error("Website ist nicht erreichbar! HTTP-Status: ${response}")
                    } else {
                        echo "Website erfolgreich erreicht! HTTP-Status: ${response}"
                    }
                }
            }
        }
    }
}
