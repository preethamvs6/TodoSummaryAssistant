# Failure and Rollback Scenarios

This document outlines the operational procedures, failure handling, and disaster recovery plans for the **Todo Summary Assistant** production deployment.

---

### Scenario 1: A faulty version is deployed. How do you roll back?

**Solution / Response:**
1. **Automated Rollback via Pipeline**: 
   * When a deployment finishes, the CI/CD pipeline immediately executes a health check against `http://<EC2_IP>:8080/actuator/health`.
   * If the health check fails or times out, the pipeline fails fast and automatically re-deploys the previous known stable Docker image tag (stored in `/home/ubuntu/.last_known_stable_backend`).
2. **Manual Rollback (Instant CLI fix)**:
   * SSH into the EC2 instance:
     ```bash
     ssh -i key.pem ubuntu@<EC2_IP>
     docker pull yourdockerhubuser/todo-backend:<PREVIOUS_TAG>
     docker rm -f todo-backend
     docker run -d --name todo-backend --restart unless-stopped --network todo-network -p 8080:8080 <PREVIOUS_IMAGE>
     ```
3. **Database Considerations**:
   * Database schema changes follow backward-compatible expansion (e.g. nullable new columns) so reverting code does not break database integrity.

---

### Scenario 2: The application crashes after deployment. What happens?

**Solution / Response:**
1. **Container Self-Healing**: 
   * Containers are configured with `--restart unless-stopped`. Docker daemon will automatically attempt to restart the crashed container.
2. **Detection & Fail-Fast in CI/CD**:
   * If the crash happens within the deployment window, the post-deployment health check (`curl /actuator/health`) fails, terminating the pipeline and triggering an immediate rollback.
3. **Alerting via Monitoring**:
   * Prometheus triggers the `ServiceDown` alert within 1 minute of container failure.
   * Grafana dashboard turns red on the "Service Up/Down Status" panel, notifying the on-call engineer.
4. **Root Cause Analysis (Logs)**:
   * View live error logs using:
     ```bash
     docker logs --tail 100 -f todo-backend
     ```

---

### Scenario 3: The CI/CD tool (Jenkins / GitHub Actions) is unavailable. Can you still deploy?

**Solution / Response:**
**Yes.** Deployments can be performed manually using the local Docker engine and SSH:
1. **Build and Tag Locally**:
   ```bash
   docker build -t yourdockerhubuser/todo-backend:v1.0.1 Backend/todo-summary-assistant
   docker build -t yourdockerhubuser/todo-frontend:v1.0.1 Frontend/todo
   ```
2. **Push to Docker Hub**:
   ```bash
   docker push yourdockerhubuser/todo-backend:v1.0.1
   docker push yourdockerhubuser/todo-frontend:v1.0.1
   ```
3. **Deploy directly on EC2 via SSH**:
   ```bash
   ssh -i key.pem ubuntu@<EC2_IP>
   docker pull yourdockerhubuser/todo-backend:v1.0.1
   docker rm -f todo-backend
   docker run -d --name todo-backend --restart unless-stopped --network todo-network -p 8080:8080 yourdockerhubuser/todo-backend:v1.0.1
   ```
4. Verify by running `curl http://localhost:8080/actuator/health`.

---

### Scenario 4: Secrets are leaked. What steps do you take?

**Solution / Response:**
1. **Immediate Revocation**:
   * **AWS Secrets/Keys**: Revoke leaked IAM access keys immediately in the AWS IAM Console.
   * **Cohere API Key**: Log in to the Cohere dashboard, revoke the compromised key, and generate a new one.
   * **Slack Webhook URL**: In Slack App Management, delete the compromised incoming webhook and generate a new URL.
   * **Database Password**: Update master password on AWS RDS via AWS CLI or Console:
     ```bash
     aws rds modify-db-instance --db-instance-identifier todo-mysql-db --master-user-password "NewStrongPassword!" --apply-immediately
     ```
2. **Update Secret Store**:
   * Update Jenkins Credentials / GitHub Secrets with the new values.
3. **Redeploy Application**:
   * Restart containers with the updated environment variables.
4. **Git Repository Hygiene**:
   * Remove leaked secrets from git commit history using `git-filter-repo` or BFG Repo-Cleaner.

---

### Scenario 5: The EC2 instance fails. How do you recover?

**Solution / Response:**
1. **Stateless Nature**:
   * The EC2 instance is completely **stateless**. No application data is stored on the EC2 instance; all persistent data lives securely in AWS RDS.
2. **Fast Recovery (AWS Console / Standard AMI)**:
   * Launch a new EC2 instance (`t3.micro`, Ubuntu 22.04) and attach the `sg-ec2` security group and `TodoApp-EC2-Role`.
   * Install Docker on the new instance:
     ```bash
     sudo apt-get update && sudo apt-get install -y docker.io
     sudo systemctl enable --now docker
     ```
   * Update the `EC2_HOST` IP in Jenkins credentials.
   * Trigger the Jenkins pipeline to automatically pull the Docker images and run the application.
3. **Re-run Pipeline**:
   * Trigger the CI/CD pipeline pointing to the new EC2 public IP to immediately deploy the latest backend and frontend containers.
4. **Production Architecture Improvement**:
   * Attach an AWS Auto Scaling Group (ASG) with a minimum size of 1 and an Application Load Balancer (ALB). If an instance fails health checks, AWS replaces it automatically with zero manual intervention.

---

### Scenario 6: The RDS database becomes unavailable. What is the impact and recovery plan?

**Solution / Response:**
* **Impact**:
  * Backend endpoints that read or write todos will fail and return HTTP 500 errors.
  * HikariCP connection pool exhausts and logs database connectivity timeout exceptions.
  * The frontend displays an error notification banner to users.
* **Recovery Plan**:
  1. **Multi-AZ Automatic Failover**:
     * In production, AWS RDS is configured with **Multi-AZ enabled**. AWS automatically fails over to the synchronous standby replica in another availability zone within 60–120 seconds with no data loss and no endpoint change.
  2. **Automated Snapshots & Point-In-Time Recovery (PITR)**:
     * AWS RDS automated daily backups and transaction logs allow restoring the database to any specific second within the retention period (up to 35 days).
  3. **Restore from Manual Snapshot**:
     ```bash
     aws rds restore-db-instance-from-db-snapshot \
       --db-instance-identifier todo-mysql-restored \
       --db-snapshot-identifier todo-backup-snapshot
     ```
     Once restored, update `DB_URL` in CI/CD secrets to point to the restored DB endpoint.
