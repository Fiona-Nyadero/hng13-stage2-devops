# hng13-stage2-devops

Stage 2 Task of the HNG Internship HG13

HNG Internship Stage 2 DevOps Task: Blue/Green DeploymentThis repository contains the solution for the Stage 2 DevOps task: deploying a Node.js service using a Blue/Green strategy behind an Nginx reverse proxy, implementing auto-failover and manual pool toggling.Architecture OverviewThe solution uses Docker Compose to orchestrate three containers:app_blue (Primary/Active): Runs the Node.js service on localhost:8081.app_green (Backup/Standby): Runs the identical Node.js service on localhost:8082.nginx (Traffic Cop): Acts as the public entry point on localhost:8080.The core logic resides in the Nginx configuration, which uses the proxy_next_upstream directive combined with tight timeouts to achieve zero-failed client requests during failover.Key Solution ComponentsFeatureImplementation DetailNginx DirectivesPrimary/BackupRoles are dynamically assigned in the nginx.conf.template via an init-nginx.sh script that reads the ACTIVE_POOL variable.server <host>:port backupAuto-FailoverBlue is marked as failed after a single error or timeout. Traffic instantly switches to Green.max_fails=1, fail_timeout=1sZero-DowntimeIf a request fails on Blue, Nginx retries the same client request to Green before reporting an error.proxy_next_upstream error timeout http_500PortsPublic entry on 8080. Direct access for chaos on 8081 (Blue) and 8082 (Green).ports: in docker-compose.ymlParameterizationAll images, release IDs, and the active pool are defined in the .env file.${VARIABLE_NAME}⚙️ How to Run the ProgramPrerequisitesYou must have Docker and Docker Compose installed on your system.1. Setup the Environment FileCopy the provided example environment file to create the file Docker Compose will automatically load.Bashcp .env.example .env 2. Start the ServicesRun Docker Compose in detached mode (-d) to start all three containers.Bashdocker-compose up -d 3. Verify ContainersEnsure all three containers are running and healthy:Bashdocker-compose ps
You should see Up status for app_blue, app_green, and nginx_proxy.🧪 Testing the Blue/Green Auto-FailoverYou can use curl to simulate client traffic and chaos to confirm the solution meets the requirements.Phase 1: Baseline (Blue Active)Confirm that traffic is correctly routed to the primary Blue service (X-App-Pool: blue).Bashcurl -s http://localhost:8080/version | grep X-App-Pool

# Expected Output: X-App-Pool: blue

Phase 2: Induce ChaosSimulate a failure on the primary service by instructing Blue to return HTTP 500 errors.Bashcurl -s -X POST http://localhost:8081/chaos/start?mode=error

# Output will confirm chaos initiated.

Phase 3: Critical Zero-Downtime FailoverImmediately send a request to the Nginx entry point. This test ensures Nginx detects the 500 from Blue and retries instantly to Green within the same client connection, resulting in a successful 200 OK for the client.Check HTTP Status Code (Must be 200):Bashcurl -s -o /dev/null -w "HTTP Code: %{http_code}\n" http://localhost:8080/version

# Expected Output: HTTP Code: 200 (Passes the zero-failure requirement)

Verify Traffic Switch:Confirm that subsequent requests are now routed directly to the Green service.Bashcurl -s http://localhost:8080/version | grep X-App-Pool

# Expected Output: X-App-Pool: green

Phase 4: Stop Chaos & Verify RecoveryStop the simulated chaos on Blue. Nginx's fail_timeout=1s ensures it quickly begins checking Blue again. The system should automatically switch back once Blue is healthy.Bash# Stop chaos on Blue
curl -s -X POST http://localhost:8081/chaos/stop

# Wait a few seconds for recovery, then check again

sleep 5
curl -s http://localhost:8080/version | grep X-App-Pool

# Expected Output: X-App-Pool: blue (Switched back to primary)

CleanupTo stop and remove all containers and networks:Bashdocker-compose down
