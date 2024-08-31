# flaskworkflow ___
---

# Flask Application with CI/CD Pipeline

This project demonstrates the deployment of a Flask application using GitHub Actions for CI/CD and systemd for managing the application as a service on an EC2 instance.

## Table of Contents

1. [Project Overview](#project-overview)
2. [Prerequisites](#prerequisites)
3. [Setting Up the Flask Application](#setting-up-the-flask-application)
4. [Creating the Python Virtual Environment](#creating-the-python-virtual-environment)
5. [Setting Up the Systemd Service](#setting-up-the-systemd-service)
6. [Configuring GitHub Actions](#configuring-github-actions)
7. [Environment Variables and Secrets](#environment-variables-and-secrets)
8. [Testing and Deployment](#testing-and-deployment)
9. [Troubleshooting](#troubleshooting)

## Project Overview

This repository contains a simple Flask application and a GitHub Actions CI/CD pipeline that automatically deploys the application to an AWS EC2 instance. The deployment is managed by systemd, which allows the Flask app to run as a service.

## Prerequisites

Before you start, ensure you have the following:

- An AWS EC2 instance running Ubuntu (or a similar Linux distribution).
- SSH access to your EC2 instance.
- A Flask application ready for deployment.
- A GitHub repository containing your Flask application.
- GitHub Secrets configured for deployment.

## Setting Up the Flask Application

1. **Clone the Repository**:

   ```bash
   git clone https://github.com/your-username/your-repo-name.git
   cd your-repo-name
   ```

2. **Install Flask**:

   If Flask is not already installed, you can install it using pip:

   ```bash
   pip install flask
   ```

3. **Create the Flask Application**:

   Ensure your Flask application is structured like this:

   ```plaintext
   your-repo-name/
   ├── app.py
   ├── requirements.txt
   ├── venv/
   └── .github/
       └── workflows/
           └── deploy.yml
   ```

   Here’s a basic `app.py` example:

   ```python
    from flask import Flask
    app= Flask(__name__)

    @app.route('/')
    def hello_world():
        return "hello ravi"

    if __name__=="__main__":
        app.run(debug=True, port=8000)
   ```

4. **Create the `requirements.txt` File**:

   List all your Python dependencies in `requirements.txt`:

   ```plaintext
   flask
   gunicorn
   ```

## Creating the Python Virtual Environment

1. **Create a Virtual Environment**:

   ```bash
   python3 -m venv venv
   ```

2. **Activate the Virtual Environment**:

   ```bash
   source venv/bin/activate
   ```

3. **Install Dependencies**:

   ```bash
   pip install -r requirements.txt
   ```

## Setting Up the Systemd Service

1. **Create the Systemd Service File**:

   Create a service file to manage your Flask application.

   ```bash
   sudo nano /etc/systemd/system/my-flask-app.service
   ```

2. **Add the Following Configuration**:

   ```ini
   [Unit]
   Description=My Flask Application
   After=network.target

   [Service]
   User=ubuntu  # Replace with your EC2 username
   WorkingDirectory=/home/ubuntu/your-repo-name  # Adjust to your app's directory
   Environment="PATH=/home/ubuntu/your-repo-name/venv/bin"
   ExecStart=/home/ubuntu/your-repo-name/venv/bin/gunicorn -w 4 -b 0.0.0.0:8000 app:app
   Restart=always

   [Install]
   WantedBy=multi-user.target
   ```

3. **Reload Systemd and Start the Service**:

   ```bash
   sudo systemctl daemon-reload
   sudo systemctl enable my-flask-app.service
   sudo systemctl start my-flask-app.service
   ```

4. **Check the Service Status**:

   ```bash
   sudo systemctl status my-flask-app.service
   ```

   You should see the service running. If there are errors, the status command will display them.

## Configuring GitHub Actions

1. **Create a Workflow File**:

   Inside your repository, create the following directory structure:

   ```plaintext
   .github/
   └── workflows/
       └── deploy.yml
   ```

2. **Define the GitHub Actions Workflow**:

   Here’s an example `deploy.yml` file:

   ```yaml
    name: Setup and Deploy

    on:
    push:
        branches:
        - main

    jobs:
    deploy:
        runs-on: ubuntu-latest

        steps:
        - name: Checkout code
        uses: actions/checkout@v3

        - name: Set up Python
        uses: actions/setup-python@v4
        with:
            python-version: '3.x'

        - name: Create SSH directory
        run: mkdir -p ~/.ssh
        

        - name: Add SSH key
        run: |
            echo "${{ secrets.EC2_SSH_KEY }}" | tee ~/.ssh/id_rsa > /dev/null
            sudo chmod 600 ~/.ssh/id_rsa

        - name: Print environment variables
        run: |
            echo "EC2_USER=${{ secrets.EC2_USER }}"
            echo "EC2_HOST=${{ secrets.EC2_HOST }}"
        
        - name: Test SSH connection
        run: |
            ssh -i ~/.ssh/id_rsa -o StrictHostKeyChecking=no ${{ secrets.EC2_USER }}@${{ secrets.EC2_HOST }} "echo SSH connection successful"
            
            
        - name: Deploy to EC2
        run: |
            rsync -avz -e "ssh -i ~/.ssh/id_rsa -o StrictHostKeyChecking=no" ./ ${{ secrets.EC2_USER }}@${{ secrets.EC2_HOST }}:${{ secrets.FLASK_APP_PATH }}
            ssh -i ~/.ssh/id_rsa -o StrictHostKeyChecking=no ${{ secrets.EC2_USER }}@${{ secrets.EC2_HOST }} << 'EOF'
            cd ${{ secrets.FLASK_APP_PATH }}
            source venv/bin/activate
            pip install -r requirements.txt
            sudo systemctl restart my-flask-app.service
            EOF
    
   ```

## Environment Variables and Secrets

To securely handle sensitive information, store them as secrets in GitHub:

1. **Add Secrets to GitHub**:

   - `EC2_SSH_KEY`: Your private SSH key.
   - `EC2_USER`: The username for SSH (e.g., `ubuntu`).
   - `EC2_HOST`: The public IP address or domain name of your EC2 instance.
   - `FLASK_APP_PATH`: The directory on the EC2 instance where the Flask app is deployed.

   To add these:

   - Go to your GitHub repository.
   - Navigate to `Settings` > `Secrets and variables` > `Actions`.
   - Add the secrets.

## Testing and Deployment

1. **Push Changes to GitHub**:

   When you push changes to the `main` or `staging` branch, the GitHub Actions workflow will automatically run and deploy the application.

2. **View Workflow Execution**:

   - Go to the `Actions` tab in your GitHub repository.
   - Click on the latest workflow run to see the logs and ensure the deployment was successful.

3. **Access the Application**:

   After a successful deployment, you can access your Flask application via the EC2 instance's public IP and port 8000:

   ```plaintext
   http://your-ec2-ip:8000
   ```

## Troubleshooting

- **Permission Denied (Publickey)**: Ensure that the correct SSH key is added to the GitHub Secrets and that the corresponding public key is in the `authorized_keys` file on the EC2 instance.
- **Service Not Starting**: Check the service status using `sudo systemctl status my-flask-app.service` to debug any issues.
- **Deployment Fails**: Review the GitHub Actions logs for details on what went wrong during deployment.
