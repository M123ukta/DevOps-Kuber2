# ISWE406P – Agile Development Process & DevOps (Assessment VIII)

One Git repository, one folder per question:

```
devops-assessment/
├── q1-lms-portal/        University Learning Management Portal   (2 replicas, NodePort 30010)
├── q2-food-portal/       Online Food Ordering Portal, Blue-Green  (3 replicas, NodePort 30020 live / 30021 preview)
├── q3-hospital-portal/   Hospital Appointment Portal              (3 replicas, NodePort 30030)
└── README.md
```

Each folder contains: `index.html`, `style.css`, `health.html`, `Dockerfile`, Kubernetes YAML(s) and a `Jenkinsfile`.

| | Image name | Deployment | Service | NodePort | Replicas |
|---|---|---|---|---|---|
| Q1 | `<user>/lms-portal` | `lms-portal` | `lms-portal-service` | **30010** | 2 |
| Q2 | `<user>/food-portal` | `food-portal-blue` / `food-portal-green` | `food-portal-live` / `food-portal-preview` | **30020** / **30021** | 3 |
| Q3 | `<user>/hospital-portal` | `hospital-portal` | `hospital-portal-service` | **30030** | 3 |

Different NodePorts mean all three can run side by side.

> **How the "faulty release" works (all three questions):**
> `docker build --build-arg FAULTY=true ...` produces an image that still builds, starts and serves the page, but has **no `/health.html`**.
> The Deployment's *readiness probe* hits `/health.html`, so the new pod never becomes Ready (probe returns 404).
> With `maxUnavailable: 0` the old pods keep serving, `kubectl rollout status` times out → the failure is detected → rollback (`kubectl rollout undo`, or traffic stays on the stable colour in Q2).

---

## 0. One-time setup

### 0.1 Replace the two placeholders
Replace `YOUR_DOCKERHUB_USERNAME` and `YOUR_GITHUB_USERNAME` everywhere.

**Git Bash / Linux / macOS**
```bash
cd devops-assessment
grep -rl "YOUR_DOCKERHUB_USERNAME\|YOUR_GITHUB_USERNAME" . --exclude-dir=.git | xargs sed -i 's/YOUR_DOCKERHUB_USERNAME/mydockerid/g; s/YOUR_GITHUB_USERNAME/mygithubid/g'
```

**PowerShell**
```powershell
cd devops-assessment
Get-ChildItem -Recurse -File | Where-Object { $_.FullName -notmatch '\\\.git\\' } | ForEach-Object {
  (Get-Content $_.FullName -Raw) -replace 'YOUR_DOCKERHUB_USERNAME','mydockerid' -replace 'YOUR_GITHUB_USERNAME','mygithubid' | Set-Content $_.FullName -NoNewline
}
```
(Files affected: the `Jenkinsfile` and `deployment*.yaml` in each folder.)

In the commands below, `mydockerid` stands for your Docker Hub username.

### 0.2 Create the GitHub repository and push
Create an empty **public** repo named `devops-assessment` on GitHub (public lets Jenkins clone without extra credentials), then:
```bash
cd devops-assessment
git init
git branch -M main
git add .
git commit -m "DevOps assessment: Q1 LMS, Q2 Food, Q3 Hospital"
git remote add origin https://github.com/mygithubid/devops-assessment.git
git push -u origin main
```

### 0.3 Enable Kubernetes in Docker Desktop
Docker Desktop → **Settings → Kubernetes → Enable Kubernetes → Apply & restart**. Wait until the Kubernetes icon is green, then:
```bash
kubectl config use-context docker-desktop
kubectl get nodes          # STATUS must be Ready
kubectl cluster-info
```

### 0.4 Log in to Docker Hub
```bash
docker login
```

### 0.5 Jenkins prerequisites
* Jenkins running on the **same machine as Docker Desktop** (Windows installer from jenkins.io, or Linux).
* Plugins: *Pipeline, Git, Credentials Binding, Timestamper* (installed by "Suggested plugins").
* The Jenkins service must be able to run `docker` and `kubectl`.
  On Windows: `services.msc` → **Jenkins** → *Log On* → run as **your own user account** → restart the service. Also make sure `docker` and `kubectl` are in the system `PATH`.
* Quick check in a Jenkins pipeline/terminal: `docker version`, `kubectl get nodes`.

### 0.6 Configure Jenkins credentials (needed by all three pipelines)
Jenkins → **Manage Jenkins → Credentials → System → Global credentials → Add Credentials**

| Kind | ID (must match exactly) | Details |
|---|---|---|
| **Username with password** | `dockerhub-creds` | Docker Hub username + password (or access token) |
| **Secret file** | `kubeconfig-creds` | Upload your kubeconfig: `C:\Users\<you>\.kube\config` (Linux/macOS: `~/.kube/config`) |

> If Jenkins runs **inside a Docker container**, it cannot see your host Docker/kubectl by default. Run Jenkins directly on the host for this assessment (recommended), or mount the Docker socket, install `docker` + `kubectl` in the container, and edit the uploaded kubeconfig so `server:` is `https://host.docker.internal:6443` (add `insecure-skip-tls-verify: true` and remove `certificate-authority-data`).

### 0.7 Create a Jenkins pipeline job (repeat once per question)
1. **New Item** → name (`lms-portal-pipeline`, `food-portal-pipeline`, `hospital-portal-pipeline`) → **Pipeline** → OK.
2. **Pipeline → Definition: Pipeline script from SCM**
   * SCM: **Git**, Repository URL: `https://github.com/mygithubid/devops-assessment.git`
   * Branch: `*/main`
   * **Script Path:** `q1-lms-portal/Jenkinsfile` (or `q2-food-portal/Jenkinsfile`, `q3-hospital-portal/Jenkinsfile`)
3. Save → **Build with Parameters** (the first run registers the parameters and the SCM-polling trigger).

---

# QUESTION 1 – University Learning Management Portal

Files: `q1-lms-portal/` → `index.html`, `style.css`, `health.html`, `Dockerfile`, `deployment.yaml`, `service.yaml`, `Jenkinsfile`

**Step 1 – Git repository** (index.html, Dockerfile, deployment.yaml): done in 0.2.
```bash
git ls-files q1-lms-portal
```

**Step 2 – Create the Docker image**
```bash
cd q1-lms-portal
docker build --build-arg APP_VERSION=v1.0 -t mydockerid/lms-portal:v1 .
docker images mydockerid/lms-portal
```

**Step 3 – Run the container and verify locally**
```bash
docker run -d --name lms-test -p 8080:80 mydockerid/lms-portal:v1
docker ps
```
Open **http://localhost:8080** (or `curl.exe -I http://localhost:8080` in PowerShell, `curl -I ...` in Bash). Then clean up:
```bash
docker logs lms-test
docker stop lms-test && docker rm lms-test
```

**Step 4 – Push to Docker Hub**
```bash
docker push mydockerid/lms-portal:v1
```
Verify at https://hub.docker.com/r/mydockerid/lms-portal/tags

**Step 5 – Enable Kubernetes in Docker Desktop:** see 0.3.

**Step 6 – Kubernetes Deployment with 2 replicas**
```bash
kubectl apply -f deployment.yaml
kubectl get deployments
kubectl rollout status deployment/lms-portal
```

**Step 7 – Jenkins credentials (Docker Hub + Kubernetes config):** see 0.6.

**Step 8 – Jenkins pipeline with all stages:** `Jenkinsfile` stages are
`Clone Code → Build Docker Image → Test Image → Push Image to Docker Hub → Deploy to Kubernetes → Health Check & Auto Rollback`.
Create the job as in 0.7 (Script Path `q1-lms-portal/Jenkinsfile`) and click **Build with Parameters** with `FAULTY_RELEASE` **unticked**.

**Step 9 – NodePort Service**
```bash
kubectl apply -f service.yaml
kubectl get svc lms-portal-service
```
(The Jenkins pipeline also applies it on every run.)

**Step 10 – Verify through the NodePort**
Open **http://localhost:30010** — the footer shows the release, e.g. *Release v3*.
```bash
curl.exe -s http://localhost:30010 | findstr Release       # PowerShell
curl -s http://localhost:30010 | grep Release              # Bash
```

**Step 11 – Both Pods Running**
```bash
kubectl get pods -l app=lms-portal
```
Expected: two pods, `READY 1/1`, `STATUS Running`.

**Every commit triggers the pipeline:** edit `q1-lms-portal/index.html`, then
```bash
git add . && git commit -m "Update LMS announcements" && git push
```
Jenkins polls GitHub every ~2 minutes (or click **Build Now**). Each build creates a new tag `v<BUILD_NUMBER>` (versioned releases).
```bash
kubectl rollout history deployment/lms-portal
```

### Q1 – Faulty release → failed rollout → automatic rollback

**A. Using Jenkins (automatic)**
1. Make sure the last build is green and the site works (stable release).
2. **Build with Parameters → tick `FAULTY_RELEASE` → Build.**
3. Watch the console: the *Health Check & Auto Rollback* stage prints `Release vN FAILED its health check. Rolling back...`, runs `kubectl rollout undo`, and the build ends **FAILED** (by design).
4. Show the evidence while/after it runs:
```bash
kubectl get pods -l app=lms-portal                       # new pod 0/1 Running (not Ready) during rollout; old pods still Running
kubectl describe pod -l app=lms-portal | findstr /i "Readiness"   # (Bash: grep -i Readiness) -> "Readiness probe failed ... 404"
kubectl rollout history deployment/lms-portal
```
5. Demonstrate the restored application:
```bash
kubectl get pods -l app=lms-portal       # 2/2 pods Running, 1/1 Ready, previous image
kubectl get deployment lms-portal -o jsonpath="{.spec.template.spec.containers[0].image}"
curl.exe -I http://localhost:30010       # HTTP/1.1 200 OK  (or open http://localhost:30010)
```

**B. Manual demo (no Jenkins)**
```bash
docker build --build-arg APP_VERSION=faulty --build-arg FAULTY=true -t mydockerid/lms-portal:faulty .
docker push mydockerid/lms-portal:faulty
kubectl set image deployment/lms-portal lms-portal=mydockerid/lms-portal:faulty
kubectl rollout status deployment/lms-portal --timeout=60s     # FAILS: timed out / exceeded progress deadline
kubectl get pods -l app=lms-portal                              # new pod not Ready, old pods still serving
kubectl rollout undo deployment/lms-portal                      # automatic rollback to previous stable release
kubectl rollout status deployment/lms-portal
kubectl get pods -l app=lms-portal
curl.exe -I http://localhost:30010
```

---

# QUESTION 2 – Online Food Ordering Portal (Automated Testing + Blue-Green)

Files: `q2-food-portal/` → `index.html`, `style.css`, `health.html`, `Dockerfile`, `deployment-blue.yaml`, `deployment-green.yaml`, `service-live.yaml`, `service-preview.yaml`, `Jenkinsfile`

**Blue-green design**
* `food-portal-blue` and `food-portal-green` are two separate Deployments (3 pods each).
* `food-portal-live` (NodePort **30020**) → real users → points to the **active** colour.
* `food-portal-preview` (NodePort **30021**) → points to the **idle** colour so the team can validate the new release first.
* Cut-over = changing the live service selector. If the new version fails, the selector never changes, so users keep reaching the previous stable version.

**Step 1 – Git repository:** done in 0.2.

**Step 2 – Build the image and test locally**
```bash
cd q2-food-portal
docker build --build-arg APP_VERSION=v1 -t mydockerid/food-portal:v1 .
docker run -d --name food-test -p 8081:80 mydockerid/food-portal:v1
```
Open **http://localhost:8081**, then
```bash
docker stop food-test && docker rm food-test
```

**Step 3 – Push to Docker Hub**
```bash
docker push mydockerid/food-portal:v1
```

**Step 4 – Kubernetes Deployment with 3 replicas (BLUE = current stable, live)**
```bash
kubectl apply -f deployment-blue.yaml
kubectl rollout status deployment/food-portal-blue
```

**Step 5 – Jenkins credentials:** see 0.6.

**Step 6 – Jenkins pipeline:** stages are
`Clone Code → Build Docker Image → Automated Tests → Push Image → Detect Active Colour → Deploy to Idle Colour → Validate on Preview → Approve & Switch Traffic → Post-Switch Verification`.
Create the job as in 0.7 (Script Path `q2-food-portal/Jenkinsfile`).

**Step 7 – NodePort Services**
```bash
kubectl apply -f service-live.yaml
kubectl apply -f service-preview.yaml
kubectl get svc
```

**Step 8 – Deploy on Docker Desktop Kubernetes** (steps 4 + 7 above).

**Step 9 – Verify Deployment and all three Pods**
```bash
kubectl get deployments
kubectl get pods -l app=food-portal -o wide      # 3 pods, READY 1/1, Running
kubectl get svc food-portal-live food-portal-preview
```
Open **http://localhost:30020** (live site, blue, release v1).

**Step 10 – Modify the HTML application (add a new feature)**
In `q2-food-portal/index.html`, replace the line `<!-- ADD-NEW-FEATURE-HERE -->` with this *Course-rating-style* **Customer Rating Summary + Thank-you message** (CSS is already in `style.css`):
```html
<section class="section tight" id="ratings">
  <div class="container">
    <h2>Customer Rating Summary</h2>
    <div class="card ratings-grid">
      <div class="big-score"><strong>4.6</strong><div>⭐⭐⭐⭐⭐</div><small>Based on 2,314 reviews</small></div>
      <div class="bars">
        <div>5★ <i><b style="width:72%"></b></i> 72%</div>
        <div>4★ <i><b style="width:18%"></b></i> 18%</div>
        <div>3★ <i><b style="width:6%"></b></i> 6%</div>
        <div>2★ <i><b style="width:3%"></b></i> 3%</div>
        <div>1★ <i><b style="width:1%"></b></i> 1%</div>
      </div>
    </div>
    <div class="thanks" style="margin-top:1.5rem">
      <h3>Thank you for ordering with FoodieHub! 💛</h3>
      <p>Your feedback helps us serve you better.</p>
    </div>
  </div>
</section>
```

**Step 11 – Build the new image**
```bash
docker build --build-arg APP_VERSION=v2 -t mydockerid/food-portal:v2 .
```

**Step 12 – Push the updated image**
```bash
docker push mydockerid/food-portal:v2
```

**Step 13 – Update the Kubernetes Deployment with the new image (blue-green way)**
Deploy v2 to the idle colour (**green**) only; live users still see blue/v1:
```bash
kubectl apply -f deployment-green.yaml                 # image already set to :v2
kubectl rollout status deployment/food-portal-green
kubectl set selector svc/food-portal-preview app=food-portal,version=green
```
Validate green at **http://localhost:30021** (new rating section is visible). When satisfied, cut over:
```bash
kubectl set selector svc/food-portal-live app=food-portal,version=green
```
*(Alternative, plain rolling update of the same deployment: `kubectl set image deployment/food-portal-blue food-portal=mydockerid/food-portal:v2`.)*

**Step 14 – Verify the updated app through the NodePort**
Open **http://localhost:30020** → shows the **Customer Rating Summary**, footer *Release v2*.
```bash
kubectl get svc food-portal-live -o jsonpath="{.spec.selector.version}"      # green
kubectl get pods -l app=food-portal
```
Instant rollback if ever needed: `kubectl set selector svc/food-portal-live app=food-portal,version=blue`

**Same flow, fully automated by Jenkins:** commit the HTML change → `git push` → **Build with Parameters** (`AUTO_PROMOTE` unticked) → pipeline builds/tests/pushes, deploys to the idle colour, validates on **http://localhost:30021**, pauses at *Approve & Switch Traffic* (click **Promote to live** in the Jenkins console), then switches live traffic and verifies.

### Q2 – Faulty release → failed rollout → traffic stays on stable version

**A. Using Jenkins**
1. **Build with Parameters → tick `FAULTY_RELEASE` → Build.**
2. *Deploy to Idle Colour* fails: `New release vN FAILED on <idle>. Live traffic is untouched and still served by <active>.` The faulty pods are scaled to 0 and the build is marked failed. The pipeline never reaches the traffic switch.
3. Evidence:
```bash
kubectl get pods -l app=food-portal                                   # faulty colour pods 0/1 (not Ready) then removed; active colour 3/3 Running
kubectl get svc food-portal-live -o jsonpath="{.spec.selector.version}"   # still the previous (stable) colour
```
4. Demonstrate restored/stable app: open **http://localhost:30020** – still the previous working release.

**B. Manual demo** (assume `green`/v2 is live, `blue` is idle)
```bash
docker build --build-arg APP_VERSION=faulty --build-arg FAULTY=true -t mydockerid/food-portal:faulty .
docker push mydockerid/food-portal:faulty
kubectl set image deployment/food-portal-blue food-portal=mydockerid/food-portal:faulty
kubectl rollout status deployment/food-portal-blue --timeout=60s      # FAILS
kubectl get pods -l app=food-portal                                    # blue pods not Ready
kubectl get svc food-portal-live -o jsonpath="{.spec.selector.version}"   # green – users unaffected
curl.exe -I http://localhost:30020                                     # 200 OK from stable green
kubectl rollout undo deployment/food-portal-blue                       # restore idle colour to its last good version
kubectl rollout status deployment/food-portal-blue
```

---

# QUESTION 3 – Online Hospital Appointment Portal (CI/CD + Rollback)

Files: `q3-hospital-portal/` → `index.html`, `style.css`, `health.html`, `Dockerfile`, `deployment.yaml`, `service.yaml`, `Jenkinsfile`

**Jenkins pipeline stages (exactly as the question lists):**
`1 Clone Code → 2 Build Docker Image → 3 Test Application → 4 Push Image → 5 Deploy to Kubernetes → Verify Rollout & Auto Rollback`

**Initial manual run (optional but good for the demo)**
```bash
cd q3-hospital-portal
docker build --build-arg APP_VERSION=v1.0 -t mydockerid/hospital-portal:v1 .
docker run -d --name hosp-test -p 8082:80 mydockerid/hospital-portal:v1     # check http://localhost:8082
docker stop hosp-test && docker rm hosp-test
docker push mydockerid/hospital-portal:v1
```

**Step 6 – Kubernetes: 3 replicas + Deployment + NodePort Service**
```bash
kubectl apply -f deployment.yaml
kubectl apply -f service.yaml
kubectl get deployments
kubectl get svc hospital-portal-service
```

**Step 7 – Jenkins credentials (Docker Hub + Kubernetes config):** see 0.6.

**Step 8 – Jenkins pipeline:** create the job as in 0.7 (Script Path `q3-hospital-portal/Jenkinsfile`) and run **Build with Parameters** with `FAULTY_RELEASE` unticked. Confirm all stages are green.

**Step 9 – Modify the webpage in GitHub** (new company/job in the placement example → here: a **new doctor** and a **new announcement**).
In `q3-hospital-portal/index.html`:

Replace `<!-- ADD-NEW-DOCTOR-HERE -->` with
```html
<article class="card doctor">
  <div class="avatar d5" aria-hidden="true">SR</div>
  <h3>Dr. Sanjay Reddy</h3>
  <p class="spec">Dermatologist</p>
  <p>MD (Dermatology) · 9 yrs experience</p>
  <span class="badge ok">New · Available today</span>
</article>
```
Replace `<!-- ADD-NEW-ANNOUNCEMENT-HERE -->` with
```html
<li><span class="tag red">New</span><div><h3>Dermatology Clinic Opens Monday</h3><p>Dr. Sanjay Reddy joins CityCare. OPD 10 AM – 2 PM, Mon – Fri.</p></div></li>
```
Commit and push (this is the GitHub modification):
```bash
git add q3-hospital-portal/index.html
git commit -m "Add new doctor and announcement"
git push
```
*(You can also edit the file directly on github.com → pencil icon → Commit changes.)*

**Step 10 – Trigger the Jenkins pipeline again**
Either wait ~2 minutes (SCM polling from the `Jenkinsfile`) or click **Build with Parameters** (FAULTY unticked).

**Step 11 – Verify Jenkins automatically does everything** (open the build's *Console Output* / Stage View):
* *Clone Code* – fetches the latest commit (console shows the new commit id)
* *Build Docker Image* – builds `mydockerid/hospital-portal:v<N>`
* *Push Image* – new tag appears on Docker Hub
* *Deploy to Kubernetes* – `deployment.apps/hospital-portal configured`
```bash
kubectl rollout history deployment/hospital-portal
kubectl get deployment hospital-portal -o jsonpath="{.spec.template.spec.containers[0].image}"
```

**Step 12 – Verify through the NodePort**
Open **http://localhost:30030** → *Dr. Sanjay Reddy* card and the new announcement are visible; footer shows the new release number.

**Step 13 – 3 Pods running**
```bash
kubectl get pods -l app=hospital-portal
```
Expected: 3 pods, `READY 1/1`, `STATUS Running`.

### Q3 – Faulty release → failed rollout → automatic rollback

**A. Using Jenkins**
1. **Build with Parameters → tick `FAULTY_RELEASE` → Build.**
2. The *Verify Rollout & Auto Rollback* stage fails the health check, logs `Rolling back to the previous stable release...`, runs `kubectl rollout undo`, waits for it to complete, then marks the build failed.
3. Evidence + restored app:
```bash
kubectl get pods -l app=hospital-portal                # during rollout: one new pod 0/1 not Ready; afterwards 3/3 Running
kubectl describe pod -l app=hospital-portal | findstr /i "Readiness"      # (Bash: grep -i Readiness)
kubectl rollout history deployment/hospital-portal
curl.exe -I http://localhost:30030                      # 200 OK – previous stable release is back
```
Open **http://localhost:30030** → the stable site (with Dr. Sanjay Reddy) is working.

**B. Manual demo**
```bash
docker build --build-arg APP_VERSION=faulty --build-arg FAULTY=true -t mydockerid/hospital-portal:faulty .
docker push mydockerid/hospital-portal:faulty
kubectl set image deployment/hospital-portal hospital-portal=mydockerid/hospital-portal:faulty
kubectl rollout status deployment/hospital-portal --timeout=60s     # FAILS
kubectl get pods -l app=hospital-portal
kubectl rollout undo deployment/hospital-portal
kubectl rollout status deployment/hospital-portal
kubectl get pods -l app=hospital-portal
curl.exe -I http://localhost:30030
```

---

## Troubleshooting

| Symptom | Fix |
|---|---|
| `docker`/`kubectl` not found in Jenkins | Add them to the system `PATH`, restart the Jenkins service; run the service as your user (0.5). |
| `denied: requested access to the resource is denied` on push | `docker login`; image name must start with **your** Docker Hub username. |
| `ImagePullBackOff` | Wrong username in `deployment*.yaml`, or the tag was never pushed / repo is private. |
| `The connection to the server ... was refused` | Kubernetes not enabled/ready in Docker Desktop; `kubectl config use-context docker-desktop`. |
| NodePort page not opening | `kubectl get svc` to confirm the port; `kubectl get pods` all `1/1 Running`; try `http://127.0.0.1:<port>`. |
| Pipeline can't find credentials | IDs must be exactly `dockerhub-creds` and `kubeconfig-creds`. |
| First pipeline run with `FAULTY_RELEASE` ticked can't roll back | There is no previous stable revision yet: run one **good** build first. |
| `Provided port is already allocated` (local test) | Use another host port, e.g. `-p 8090:80`. |

## Clean-up (after the demo)
```bash
kubectl delete -f q1-lms-portal/deployment.yaml -f q1-lms-portal/service.yaml
kubectl delete -f q2-food-portal/deployment-blue.yaml -f q2-food-portal/deployment-green.yaml -f q2-food-portal/service-live.yaml -f q2-food-portal/service-preview.yaml
kubectl delete -f q3-hospital-portal/deployment.yaml -f q3-hospital-portal/service.yaml
```
