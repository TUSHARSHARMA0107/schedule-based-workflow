# schedule-based-workflow



name: CI/CD Devops Pipeline

on:
  push:
    branches: [ main ] # Runs automatically when you push to the main branch

jobs:
  # JOB 1: Compile, build, and test your code
  build-and-test:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout Code
        uses: actions/checkout@v4

      - name: Setup Node.js Environment
        uses: actions/setup-node@v4
        with:
          node-version: '20'

      - name: Install Dependencies and Build
        run: |
          npm install
          npm run build --if-present

  # JOB 2: Package the built application into a Docker Image
  package-docker:
    needs: build-and-test # Only runs if the build job finishes successfully
    runs-on: ubuntu-latest
    steps:
      - name: Checkout Code
        uses: actions/checkout@v4

      - name: Log in to Docker Hub
        uses: docker/login-action@v3
        with:
          username: ${{ vars.DOCKER_USERNAME }}
          password: ${{ secrets.DOCKER_PASSWORD }}

      - name: Build and Push Docker Image
        uses: docker/build-push-action@v6
        with:
          context: .
          push: true
          # Tags the image as 'latest' under your account
          tags: ${{ vars.DOCKER_USERNAME }}/${{ vars.DOCKER_REPO_NAME }}:latest

  # JOB 3: Tell your live server to download and run the new Docker container
  deploy-to-server:
    needs: package-docker # Only runs if the Docker image is successfully pushed
    runs-on: ubuntu-latest
    steps:
      - name: Execute Remote SSH Commands on Server
        uses: appleboy/ssh-action@master
        with:
          host: ${{ vars.SERVER_HOST }}
          username: ${{ vars.SERVER_USER }}
          key: ${{ secrets.SERVER_SSH_KEY }}
          script: |
            # 1. Log into Docker Hub on your target server
            docker login -u "${{ vars.DOCKER_USERNAME }}" -p "${{ secrets.DOCKER_PASSWORD }}"
            
            # 2. Pull the brand new image you just created in Job 2
            docker pull ${{ vars.DOCKER_USERNAME }}/${{ vars.DOCKER_REPO_NAME }}:latest
            
            # 3. Stop and remove the old running application container (ignoring errors if it doesn't exist yet)
            docker stop my-running-app || true
            docker rm my-running-app || true
            
            # 4. Start the new container on port 80
            docker run -d --name my-running-app -p 80:80 ${{ vars.DOCKER_USERNAME }}/${{ vars.DOCKER_REPO_NAME }}:latest
