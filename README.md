# CORS Troubleshooting and Resolution – Student Management Project

## 1. Problem Overview

While deploying the Student Management application on Minikube, the React frontend was unable to communicate with the Spring Boot backend.

The frontend was accessible, and the backend Pods were running, but API requests from the browser were failing because of a CORS (Cross-Origin Resource Sharing) error.

### Original Browser Error

    Access to fetch at 'http://192.168.49.2:30978/api/students'
    from origin 'http://192.168.49.2:30080' has been blocked by CORS policy:

    Response to preflight request doesn't pass access control check:
    No 'Access-Control-Allow-Origin' header is present.

The backend also returned:

    HTTP/1.1 403
    Invalid CORS request

## 2. Application Architecture

The application was deployed using Minikube on Fedora.

| Component | Technology | Address |
|---|---|---|
| Frontend | React + TypeScript + Vite | http://192.168.49.2:30080 |
| Web Server | Nginx | Frontend container |
| Backend | Spring Boot | Container port 8082 |
| Backend Service | Kubernetes NodePort | http://192.168.49.2:30978 |
| Containerization | Docker | Backend and frontend images |
| Orchestration | Kubernetes / Minikube | Local cluster |
| Backend API | REST API | /api/students |

### Kubernetes Resources

- Backend Deployment: student-management-deployment
- Backend Service: student-management-service
- Frontend NodePort: 30080
- Backend NodePort: 30978
- Minikube IP: 192.168.49.2

### Communication Flow

    Browser
       |
       v
    React Frontend
    192.168.49.2:30080
       |
       | HTTP Request
       v
    Kubernetes NodePort
    192.168.49.2:30978
       |
       v
    Backend Service
       |
       v
    Spring Boot Pods
       |
       v
    REST API
    /api/students

## 3. Understanding the CORS Problem

CORS stands for Cross-Origin Resource Sharing.

Browsers enforce the Same-Origin Policy, which restricts frontend JavaScript from accessing resources hosted on a different origin unless the server explicitly allows it.

An origin consists of:

- Protocol (HTTP or HTTPS)
- Hostname or IP address
- Port number

In this application:

    Frontend: http://192.168.49.2:30080
    Backend:  http://192.168.49.2:30978

Although both use the same IP address, they use different ports.

Therefore, they are different origins.

### What Is a Preflight Request?

Before sending certain cross-origin requests, the browser sends an HTTP OPTIONS request to check whether the backend allows the requested operation.

For example:

    OPTIONS /api/students
    Origin: http://192.168.49.2:30080
    Access-Control-Request-Method: GET
    Access-Control-Request-Headers: content-type

The backend must respond with appropriate CORS headers.

A successful response looks like:

    HTTP/1.1 200
    Access-Control-Allow-Origin: http://192.168.49.2:30080
    Access-Control-Allow-Methods: GET,POST,PUT,DELETE,OPTIONS
    Access-Control-Allow-Headers: content-type

If the preflight request fails, the browser blocks the actual API request.

This was the reason the React frontend could not retrieve student records.

## 4. Root Causes Identified

### Issue 1: CORS Configuration Was Missing from the Packaged JAR

The project already contained:

    backend/src/main/java/com/example/studentmanagement/CorsConfig.java

The configuration source code was:

    package com.example.studentmanagement;

    import org.springframework.context.annotation.Bean;
    import org.springframework.context.annotation.Configuration;
    import org.springframework.web.servlet.config.annotation.CorsRegistry;
    import org.springframework.web.servlet.config.annotation.WebMvcConfigurer;

    @Configuration
    public class CorsConfig {

        @Bean
        public WebMvcConfigurer corsConfigurer() {
            return new WebMvcConfigurer() {
                @Override
                public void addCorsMappings(CorsRegistry registry) {
                    registry.addMapping("/api/**")
                            .allowedOrigins(
                                    "http://localhost:5173",
                                    "http://192.168.49.2:30080",
                                    "https://student-mgmt.rohitnet.dpdns.org"
                            )
                            .allowedMethods("GET", "POST", "PUT", "DELETE", "OPTIONS")
                            .allowedHeaders("*");
                }
            };
        }
    }

However, when the existing JAR was inspected:

    jar tf target/student-management-0.0.1-SNAPSHOT.jar | grep CorsConfig

There was no output.

This indicated that the packaged JAR did not contain the CORS configuration class.

**Effect:**

Spring Boot was running without the intended CORS configuration.

Consequently, the backend rejected the browser's preflight request with HTTP 403.

### Issue 2: The Backend JAR Was Not Rebuilt Correctly

The source code and the compiled application were not synchronized.

Changes made to Java source files do not automatically modify an already generated JAR.

The old JAR had to be removed and rebuilt.

The following command was executed from the backend directory:

    mvn clean package -DskipTests

Explanation:

| Command | Purpose |
|---|---|
| mvn | Executes Maven |
| clean | Removes previous build output |
| package | Compiles and packages the application |
| -DskipTests | Skips test execution |

After rebuilding, the JAR was inspected again:

    jar tf target/student-management-0.0.1-SNAPSHOT.jar | grep CorsConfig

This time, the following classes appeared:

    BOOT-INF/classes/com/example/studentmanagement/CorsConfig$1.class
    BOOT-INF/classes/com/example/studentmanagement/CorsConfig.class

This confirmed that the new JAR contained the CORS configuration.

### Issue 3: Kubernetes Was Running an Old Docker Image

Initially, the Kubernetes Deployment was using:

    callmee6/student-management:1.0

A new Docker image was built, but the running Pods were still associated with the older image digest.

The old Pods showed an image digest beginning with:

    sha256:1421d370...

The newly built image had a different manifest digest:

    sha256:b17aa263257801ce0103774226f95ce5e62505cd1358da76a1c0bd8807c76b2b

These different digests indicated that the old and new images were not identical.

**Effect:**

Even after rebuilding the JAR, Kubernetes continued running an older application image.

This is why the CORS problem persisted after the Java source code was corrected.

Building a Docker image does not automatically update an existing Kubernetes Deployment or its running Pods.

### Issue 4: Image Pull Policy and Minikube Image Availability

The original Deployment used:

    imagePullPolicy: Always

However, the new image was being built directly inside Minikube.

This introduced a difference between the image available inside Minikube and the image Kubernetes attempted to retrieve from a registry.

For this local Minikube workflow, the Deployment was changed to:

    imagePullPolicy: Never

This instructs Kubernetes to use the image already available locally and not attempt to pull it from a registry.

**Important:** This configuration is appropriate for the local Minikube workflow used in this troubleshooting session. It is not a universal production deployment setting.

With Never, the required image must already exist inside Minikube.

## 5. Solution: Step-by-Step Resolution

### Step 1: Rebuild the Backend JAR

Navigate to the backend directory:

    cd ~/Downloads/student-management/backend

Build the application:

    mvn clean package -DskipTests

Verify that the JAR contains the CORS configuration:

    jar tf target/student-management-0.0.1-SNAPSHOT.jar | grep CorsConfig

Expected output:

    BOOT-INF/classes/com/example/studentmanagement/CorsConfig$1.class
    BOOT-INF/classes/com/example/studentmanagement/CorsConfig.class

### Step 2: Build a New Docker Image Inside Minikube

Navigate to the backend directory if necessary:

    cd ~/Downloads/student-management/backend

Build the new image:

    minikube image build -t callmee6/student-management:1.1 .

This builds the backend Docker image directly inside the Minikube environment.

A new image tag was used instead of reusing 1.0.

### Step 3: Update the Kubernetes Deployment Image

Update the backend Deployment:

    kubectl set image deployment/student-management-deployment \
      student-management=callmee6/student-management:1.1

This changes the image referenced by the Deployment.

### Step 4: Change the Image Pull Policy

Since the image was built inside Minikube, update the pull policy:

    kubectl patch deployment student-management-deployment \
      -p '{"spec":{"template":{"spec":{"containers":[{"name":"student-management","imagePullPolicy":"Never"}]}}}}'

Now Kubernetes uses the image already available in Minikube.

### Step 5: Verify the Deployment Rollout

Execute:

    kubectl rollout status deployment/student-management-deployment

Expected output:

    deployment "student-management-deployment" successfully rolled out

Check the Pods:

    kubectl get pods -l app=student-management

Both backend Pods were running successfully:

    student-management-deployment-65c4846df-4l772   1/1   Running
    student-management-deployment-65c4846df-p9md4   1/1   Running

### Step 6: Verify Backend CORS Using Port Forwarding

Port forwarding allows direct testing of the backend without depending on the NodePort.

Run:

    kubectl port-forward deployment/student-management-deployment 18082:8082

Keep this terminal running.

Open another terminal and execute:

    curl -i -X OPTIONS http://localhost:18082/api/students \
      -H "Origin: http://192.168.49.2:30080" \
      -H "Access-Control-Request-Method: GET" \
      -H "Access-Control-Request-Headers: content-type"

Before the correction, the backend returned:

    HTTP/1.1 403
    Invalid CORS request

After the correction, the backend returned:

    HTTP/1.1 200
    Access-Control-Allow-Origin: http://192.168.49.2:30080
    Access-Control-Allow-Methods: GET,POST,PUT,DELETE,OPTIONS
    Access-Control-Allow-Headers: content-type
    Access-Control-Max-Age: 1800

This confirmed that the backend's CORS configuration was now working when accessed through port forwarding.

## 6. Final Verification

The following checks should be performed after deployment.

### Check 1: Backend API

With port forwarding running:

    curl -i http://localhost:18082/api/students

This verifies that the backend API responds independently of the frontend.

### Check 2: Kubernetes NodePort CORS

    curl -i -X OPTIONS http://192.168.49.2:30978/api/students \
      -H "Origin: http://192.168.49.2:30080" \
      -H "Access-Control-Request-Method: GET" \
      -H "Access-Control-Request-Headers: content-type"

Expected result:

    HTTP/1.1 200
    Access-Control-Allow-Origin: http://192.168.49.2:30080

### Check 3: Frontend

Open:

    http://192.168.49.2:30080/students

Refresh the page.

Verify that:

- The Students page loads.
- The API request succeeds.
- Student records are displayed.
- No CORS errors appear in the browser console.

Use Developer Tools:

    F12 → Network → Fetch/XHR
    F12 → Console

Check the OPTIONS and GET requests.

**Note:** The direct port-forward CORS test passed. The NodePort and final browser workflow must also be verified independently before declaring the entire frontend-to-backend communication fixed.

## 7. Important Kubernetes Commands for Future Troubleshooting

| Command | Purpose |
|---|---|
| kubectl get pods -o wide | View Pods, status and IP addresses |
| kubectl get deployments | View Deployments |
| kubectl get services | View Services |
| kubectl get endpoints student-management-service | Check backend Service endpoints |
| kubectl describe deployment student-management-deployment | Inspect Deployment configuration |
| kubectl describe pod <pod-name> | Inspect a specific Pod |
| kubectl logs <pod-name> | View application logs |
| kubectl rollout status deployment/student-management-deployment | Check rollout status |
| kubectl rollout history deployment/student-management-deployment | View Deployment history |
| kubectl rollout restart deployment/student-management-deployment | Restart Pods |
| kubectl get pods --show-labels | Inspect Pod labels |

## 8. Best Practices to Avoid This Problem in the Future

### 8.1 Always Rebuild the JAR After Backend Changes

Do not assume that modifying Java source code automatically updates the packaged application.

Use:

    mvn clean package -DskipTests

### 8.2 Verify Important Classes Inside the JAR

For example:

    jar tf target/student-management-0.0.1-SNAPSHOT.jar | grep CorsConfig

This helps identify whether the required class was actually packaged.

### 8.3 Build the Docker Image After Building the JAR

The correct order is:

    Java source changes
           |
           v
    Maven clean package
           |
           v
    Fresh JAR
           |
           v
    Docker image build
           |
           v
    Kubernetes Deployment update

Do not build the Docker image before generating the latest JAR.

### 8.4 Use Versioned Docker Image Tags

Avoid repeatedly overwriting the same tag.

Prefer:

    callmee6/student-management:1.0
    callmee6/student-management:1.1
    callmee6/student-management:1.2
    callmee6/student-management:1.3

This makes it easier to identify which application version is deployed.

### 8.5 Understand Image Pull Policies

| Policy | Behaviour |
|---|---|
| Always | Attempts to pull the image from a registry when starting a container |
| IfNotPresent | Uses a local image if available; otherwise pulls it |
| Never | Uses only the local image; fails if unavailable |

For the current local Minikube workflow:

    imagePullPolicy: Never

is suitable when the image is built directly inside Minikube.

For a registry-based deployment, use an appropriate registry and pull policy instead.

### 8.6 Verify the Running Image

Inspect the Deployment:

    kubectl describe deployment student-management-deployment

Inspect the Pod image:

    kubectl get pods -l app=student-management \
      -o jsonpath='{range .items[*]}{.metadata.name}{" "}{.spec.containers[0].image}{"\n"}{end}'

Remember:

A successful Docker build does not prove that Kubernetes is running that image.

### 8.7 Test the Backend Before the Frontend

Follow this order:

1. Test the backend directly.
2. Test the CORS preflight request.
3. Test the Kubernetes NodePort.
4. Test the frontend in the browser.

This isolates the failing layer and avoids unnecessary changes.

### 8.8 Configure CORS Carefully

- Allow the exact frontend origin.
- Include the required HTTP methods.
- Include the required headers.
- Configure CORS correctly if Spring Security is introduced.
- Avoid unrestricted origins in production.
- Remember that CORS does not replace authentication or authorization.

## 9. Troubleshooting Checklist

Before reporting another CORS or API deployment issue, verify the following:

- [ ] Is the CORS configuration present in the Java source?
- [ ] Did Maven successfully rebuild the application?
- [ ] Does the JAR contain CorsConfig.class?
- [ ] Was the Docker image built after the JAR?
- [ ] Was the image built in the correct environment?
- [ ] Did I use a new image tag?
- [ ] Does the Deployment reference the new image tag?
- [ ] Is the image pull policy appropriate?
- [ ] Did the Kubernetes rollout complete?
- [ ] Are all expected backend Pods ready?
- [ ] Does the backend API return a valid response?
- [ ] Does the OPTIONS request return HTTP 200 with the correct CORS headers?
- [ ] Does the NodePort respond correctly?
- [ ] Does the browser successfully retrieve the student records?

## 10. Key Lessons Learned

This troubleshooting session demonstrated that a deployment consists of multiple independent stages.

| Stage | What it contains |
|---|---|
| Source Code | Java files and configuration |
| Build Artifact | Compiled Spring Boot JAR |
| Container Image | Docker image containing the JAR |
| Kubernetes Deployment | Desired image and Pod configuration |
| Kubernetes Pod | Actual running application container |

A change in one stage does not automatically propagate to every subsequent stage.

The most important lesson from this incident:

**Always verify the actual application running inside Kubernetes instead of assuming that the latest source code or Docker image is already deployed.**

The CORS configuration was present in the source code, but the old packaged application and old container image prevented that configuration from taking effect.

Rebuilding the JAR, building a new Minikube image, updating the Deployment and verifying the preflight response resolved the backend CORS configuration issue.

## 11. Quick Reference: Complete Backend Redeployment Workflow

For future backend code changes, use this sequence:

    # 1. Navigate to backend
    cd ~/Downloads/student-management/backend

    # 2. Build a fresh Spring Boot JAR
    mvn clean package -DskipTests

    # 3. Build a new Docker image inside Minikube
    minikube image build -t callmee6/student-management:1.2 .

    # 4. Update Kubernetes Deployment
    kubectl set image deployment/student-management-deployment \
      student-management=callmee6/student-management:1.2

    # 5. Wait for deployment
    kubectl rollout status deployment/student-management-deployment

    # 6. Check backend Pods
    kubectl get pods -l app=student-management

    # 7. Check Deployment image
    kubectl describe deployment student-management-deployment

    # 8. Check backend Service
    kubectl get svc student-management-service

    # 9. Check Service endpoints
    kubectl get endpoints student-management-service

**Important:** Change the image tag to the next version for each new build. Since the current Deployment uses imagePullPolicy: Never, always build the new image inside Minikube before updating the Deployment.
