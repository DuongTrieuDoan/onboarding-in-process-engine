# **Camunda 7 and Colima Setup Operations and Troubleshooting Guide**

A complete operational runbook covering the installation, configuration, error recovery, and runtime management of Camunda 7 Community Edition on macOS using Colima and Docker CLI.

## **1\. Prerequisites and Environment Cleanup**

When Docker Desktop is blocked or flagged by macOS Gatekeeper or corporate security policies, remove any leftover helper daemons and GUI binaries before setting up the standalone runtime:  
sudo pkill \-9 \-f '\[dD\]ocker'  
sudo launchctl bootout system /Library/LaunchDaemons/com.docker.vmnetd.plist  
sudo launchctl bootout system /Library/LaunchDaemons/com.docker.socket.plist  
sudo rm \-rf /Library/PrivilegedHelperTools/com.docker.\*  
sudo rm \-rf /Library/LaunchDaemons/com.docker.\*  
sudo rm \-rf /Applications/Docker.app

## **2\. Installing Colima and Docker CLI**

Install the open-source container engine and command-line tools via Homebrew:  
brew install colima docker docker-compose  
Fix directory permissions if Docker Desktop left root-owned directories:  
sudo chown \-R \$(whoami):admin /usr/local/lib/docker  
chmod \-R u+w /usr/local/lib/docker  
brew link docker-compose

## **3\. Starting and Configuring Colima**

Start Colima with recommended resource allocation (Native Apple Silicon ARM64):  
colima start \--cpu 2 \--memory 4  
Set persistent environment variables and socket symlinks:  
export DOCKER\_HOST="unix://\${HOME}/.colima/default/docker.sock"  
echo 'export DOCKER\_HOST="unix://\${HOME}/.colima/default/docker.sock"' \>\> \~/.zshrc  
sudo ln \-sf \~/.colima/default/docker.sock /var/run/docker.sock

## **4\. Deploying and Operating Camunda 7**

Launch the container on port 9090 (or 8081):  
docker run \-d \--name camunda \-p 9090:8080 camunda/camunda-bpm-platform:run-latest  
Runtime Management Commands:

> * Stream live logs: docker logs \-f camunda (Press Ctrl \+ C to exit without stopping container)  
> * Verify container status: docker ps  
> * Stop Camunda: docker stop camunda  
> * Restart Camunda: docker start camunda  
> * Remove Camunda: docker rm \-f camunda

Access Endpoints:

> * Web Application: http\://localhost:9090 (Username: demo / Password: demo)  
> * Engine REST API: http\://localhost:9090/engine-rest

## **5\. Troubleshooting Runbook**

Error 1: Malware Blocked: Docker.app was not opened / Killed: 9  
Fix: sudo rm \-rf /Applications/Docker.app && brew install docker  
Error 2: error provisioning docker: signal: killed (x86\_64 emulation crash)  
Fix: colima delete \-f && colima start \--cpu 2 \--memory 4  
Error 3: Permission denied @ apply2files \- /usr/local/lib/docker/cli-plugins  
Fix: sudo chown \-R \$(whoami):admin /usr/local/lib/docker && brew link docker-compose  
Error 4: context "colima": context not found  
Fix: docker context use default && rm \-rf \~/.docker/contexts && colima restart  
Error 5: docker-credential-desktop not found in PATH  
Fix: rm \~/.docker/config.json  
Error 6: failed to connect to unix:///var/run/docker.sock  
Fix: export DOCKER\_HOST="unix://\${HOME}/.colima/default/docker.sock" && sudo ln \-sf \~/.colima/default/docker.sock /var/run/docker.sock

## **6\. Daily Workflow**

> 1. Start: colima start && docker start camunda  
> 2. Verify: docker ps (visit http\://localhost:9090)  
> 3. Stop: docker stop camunda && colima stop