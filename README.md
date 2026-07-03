# DevSecOps CI/CD Security Lab — Aqua, Clair, Snyk

## 11.1 Objective

- Understand the principles of DevSecOps and the integration of security practices within the DevOps pipeline.
- Learn about the best practices and tools used for enhancing security in CI/CD pipelines.
- Gain hands-on experience implementing security checks in CI/CD pipelines.

## Resources Used

- Computer with internet access
- Docker
- CI/CD tool: Jenkins (primary) / GitLab CI (alternative config included)
- Security tools: Aqua Security, Clair, Snyk (Trivy used as a fallback — see Section 6)

## Project Structure

```
devsecops-lab/
├── app/
│   ├── package.json
│   ├── server.js
│   └── Dockerfile
├── Jenkinsfile
├── .gitlab-ci.yml
├── clair-config.yaml
└── README.md
```

---

## 11.2 Task 1: Introduction to DevSecOps

**DevSecOps** integrates security practices throughout the entire DevOps lifecycle rather than treating
security as a final gate before release. It follows the **shift-left** approach — addressing
vulnerabilities as early as the development and build stages, rather than after deployment,
when fixes are far more expensive and risky.

**Key Principles:**
- Collaboration between development, security, and operations teams.
- Automation of security tasks within the CI/CD pipeline (no manual gatekeeping).
- Continuous monitoring and feedback loops so issues are caught on every commit, not just at release.

---

## 11.3 Task 2: Best Practices and Tools for DevSecOps

**Best Practices**
- Implement security at every stage of the DevOps pipeline (build, test, deploy).
- Use automated tools for static and dynamic code analysis.
- Conduct regular security training and awareness programs for the team.

**Security Tools Used**

| Tool | Purpose |
|------|---------|
| **Aqua Security** | Container security — image scanning and runtime protection |
| **Clair** | Open-source static analysis of vulnerabilities in application containers (e.g. Docker) |
| **Snyk** | Developer-first platform to find and fix vulnerabilities in dependencies, container images, and Kubernetes applications |

---

## 11.4 Task 3: Lab — Implementing Security Checks in CI/CD Pipelines

### Setting Up the Environment

```bash
# Install Docker
sudo apt update
sudo apt install docker.io -y
sudo systemctl start docker

# Install Node.js (for the sample app)
sudo apt install nodejs npm -y
```

### Sample Application

The `app/` folder contains a minimal Node.js/Express app. It intentionally pins slightly older
dependency versions (`express@4.17.1`, `lodash@4.17.19`) so the security scanners have real,
demonstrable CVEs to detect — this makes for a much more convincing lab demo than a clean app
with nothing to find.

### Integrating Security Checks

**1. Aqua Security**
```bash
# Install Aqua CLI scanner
curl -sfL https://get.aquasec.com/scanner | sh

# Run local scan against your built image
docker build -t my-app ./app
aqua scan --local my-app
```

**2. Clair**
```bash
# Install Clair + clairctl companion tool
# (Requires a running Postgres instance — see clair-config.yaml)
clairctl analyze my-app
```

**3. Snyk**
```bash
# Install Snyk CLI
npm install -g snyk

# Authenticate (opens browser — free account works)
snyk auth

# Scan dependencies
cd app
snyk test

# Scan the built Docker image
snyk test --docker my-app --file=Dockerfile
```

Each tool is wired into the `Jenkinsfile` as its own pipeline stage (see `Jenkinsfile` in this repo),
so every build automatically triggers all three scans, and the pipeline **fails on critical vulnerabilities**
via the `Fail on Critical Vulnerabilities` stage.

### Testing and Verification

1. Trigger the CI/CD pipeline (push to `main`, or run Jenkins job manually).
2. Review scan output in the Jenkins console log for each stage.
3. Fix a vulnerability and re-run to demonstrate the feedback loop:
   ```bash
   cd app
   npm audit fix
   docker build -t my-app .
   snyk test --docker my-app --file=Dockerfile
   ```
4. Confirm the pipeline **fails** the build when a critical CVE is present, and **passes** after remediation.

---

## 11.5 Task 4: Reflection and Discussion

- Integrating security checks directly into CI/CD forces vulnerabilities to be caught at build time,
  not after deployment — this is the practical meaning of "shift-left."
- Automated tools remove reliance on manual review, which doesn't scale and is easy to skip under
  deadline pressure.
- Aqua and Clair focus on container/image-layer vulnerabilities; Snyk adds dependency-level scanning
  with actionable fix suggestions, which is why using more than one tool gives broader coverage.
- A pipeline that fails the build on critical vulnerabilities creates a real enforcement mechanism,
  not just a report nobody reads.

---

## 11.6 Deliverables

- ✅ CI/CD pipeline configuration files — `Jenkinsfile`, `.gitlab-ci.yml`
- ✅ Security scan reports — generate with:
  ```bash
  snyk test --json > snyk-report.json
  trivy image -f json -o trivy-report.json my-app
  ```
- ✅ Evidence of fixed vulnerabilities and improved security posture — run `npm audit fix`,
  rebuild, and rescan; compare the before/after vulnerability counts from the JSON reports.

---

## 6. Fallback Plan (if Aqua/Clair can't be installed in time)

Aqua and Clair both require extra infrastructure (Aqua needs a licensed server component for
full features; Clair needs a Postgres database). If you're short on time before your practical,
**Trivy** is a fully open-source, single-binary scanner that does the same job and is safe to
mention as a substitute:

```bash
sudo apt install wget apt-transport-https gnupg lsb-release -y
wget -qO - https://aquasecurity.github.io/trivy-repo/deb/public.key | sudo apt-key add -
echo "deb https://aquasecurity.github.io/trivy-repo/deb $(lsb_release -sc) main" | sudo tee -a /etc/apt/sources.list.d/trivy.list
sudo apt update
sudo apt install trivy -y

trivy image my-app
```

If asked in your viva: *"I used Trivy as a lightweight open-source alternative to Clair/Aqua due to
setup/licensing constraints — it applies the same shift-left, static vulnerability scanning principle
against the container image."* This is an honest, defensible answer examiners generally accept.
