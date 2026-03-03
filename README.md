# Docker Image Specification: FairCom Edge Suite

**Owner:** Product Management  
**Last Updated:** March 2026  
**Status:** Draft  
**Scope:** FairCom Edge Suite (extensible to other FairCom products)

---

## Purpose

Docker images are a **product delivery mechanism**, not an engineering artifact. As such, image composition, size constraints, included components, and public-facing documentation fall under Product Management ownership. Engineering implements to this specification.

This document defines requirements for FairCom Edge Suite Docker images. Once validated, this specification will serve as the template for containerizing other FairCom products (FairCom DB, RTG).

---

## Ownership and Governance

| Aspect | Owner | Responsibility |
|--------|-------|----------------|
| Image composition | Product | Define what ships |
| Size budget | Product | Set and enforce limits |
| Base image selection | Product | Approve distro choices |
| Docker Hub repository | Product | Own account, namespace, settings |
| Docker Hub README | Product + Marketing | Write and approve all copy |
| Docker Hub tags/releases | Product | Approve which tags go public |
| Build implementation | Engineering | Execute to spec |
| CI/CD pipeline | Engineering | Build automation per Product requirements |
| Security scanning | Engineering | Report findings to Product |

Product approves all changes to image contents and public documentation before release.

---

## Docker Hub Repository Requirements

### Account and Namespace

FairCom maintains an official Docker Hub organization account.

| Setting | Value |
|---------|-------|
| Organization | `faircom` |
| Repository naming | `faircom/<product>` |
| Account credentials | Owned by Product (not individual engineers) |
| 2FA | Required for all users with push access |

**Initial repository:**
- `faircom/edge-suite`

**Future repositories (when containerized):**
- `faircom/faircom-db`
- `faircom/rtg`

### Repository Settings

Each repository must be configured as follows:

- **Visibility:** Public
- **Publisher verification:** Enabled (Docker Verified Publisher if eligible)
- **Content trust:** Enabled for image signing
- **Vulnerability scanning:** Enabled via Docker Scout

### Access Control

| Role | Access Level | Assigned To |
|------|--------------|-------------|
| Owner | Full administrative | Product Director |
| Admin | Manage settings, users | Product Manager(s) |
| Write | Push images | CI/CD service account only |
| Read | Pull images | Public |

**No individual engineer accounts receive write access.** All pushes occur through automated pipelines using service account credentials managed by Product.

---

## Documentation Requirements

All public-facing documentation is owned by Product and Marketing. Engineering does not publish documentation without approval.

### Docker Hub README

The README displayed on Docker Hub is marketing collateral, not technical notes. It must:

1. Follow FairCom brand guidelines
2. Be written/approved by Product and Marketing
3. Live in a Product-controlled repository (not the application source repo)
4. Deploy automatically via pipeline when updated

**Required sections:**

| Section | Owner | Content |
|---------|-------|---------|
| Product description | Marketing | Value proposition, use cases |
| Quick start | Product | Minimal working example |
| Configuration reference | Product | Environment variables, volumes, ports |
| Support and resources | Marketing | Links to docs, community, sales |
| License | Legal | License summary and link |

**Prohibited content:**
- Engineering jargon without explanation
- Internal ticket references
- Incomplete or placeholder text
- Unapproved third-party links

### README Source Management

```
repo: faircom/docker-hub-docs (Product-owned)
├── edge-suite/
│   ├── README.md
│   └── metadata.json
├── faircom-db/           # Future
│   ├── README.md
│   └── metadata.json
└── rtg/                  # Future
    ├── README.md
    └── metadata.json
```

Changes to README files require:
1. Pull request with Product or Marketing approval
2. Merge triggers automated push to Docker Hub

---

## Automated Build Pipeline

### Architecture

```
┌─────────────────┐      ┌─────────────────┐      ┌─────────────────┐
│  Source Repo    │      │   CI Platform   │      │   Docker Hub    │
│  (Engineering)  │─────▶│  (GitHub/GitLab)│─────▶│   (Product)     │
└─────────────────┘      └─────────────────┘      └─────────────────┘
        │                        │                        │
   Tag/Release              Build & Test              Push Image
   (Product approved)       Scan, Validate            Update README
```

### Pipeline Requirements

| Requirement | Implementation |
|-------------|----------------|
| Trigger | Git tag matching release pattern (e.g., `v*.*.*`) |
| Build environment | Ephemeral, reproducible runners |
| Credentials | Docker Hub service account token (rotated quarterly) |
| Image signing | Content trust enabled, keys managed by Product |
| Vulnerability scan | Fail build on critical/high CVEs |
| Size validation | Fail build if image exceeds size budget |
| README sync | Separate pipeline from Product-owned docs repo |

### Webhook Configuration

Docker Hub webhooks notify downstream systems of new image availability.

**Required webhooks:**

| Event | Target | Purpose |
|-------|--------|---------|
| Image push | Internal notification system | Alert Product of new release |
| Vulnerability found | Security team endpoint | Trigger remediation workflow |

**Webhook security:**
- All endpoints use HTTPS
- Webhook secrets stored in vault, not source control
- Payload validation required on receiving end

### Tag Strategy

| Tag Pattern | Meaning | Automated |
|-------------|---------|-----------|
| `latest` | Most recent stable release | Yes |
| `X.Y.Z` | Specific version (immutable) | Yes |
| `X.Y` | Latest patch for minor version | Yes |
| `X` | Latest minor for major version | Yes |
| `edge` | Latest development build | Yes (from main branch) |
| `rc-X.Y.Z` | Release candidate | Yes (from RC tag) |

**Tag governance:**
- Version tags are immutable once pushed
- `latest` updates only on stable releases (not RCs)
- Product approves promotion from `edge` to version tags

### Release Workflow

1. **Engineering** completes feature work, requests release
2. **Product** reviews changelog, approves release
3. **Product** creates signed Git tag in source repo
4. **Pipeline** triggers automatically on tag
5. **Pipeline** builds, tests, scans image
6. **Pipeline** pushes to Docker Hub (if all checks pass)
7. **Pipeline** syncs README from docs repo
8. **Webhook** notifies Product of successful publish
9. **Product** verifies Docker Hub listing, announces release

---

## Size Requirements

**Target:** Images must be as small as operationally viable.

| Image Type | Maximum Size |
|------------|--------------|
| Runtime image | 75 MB compressed |
| Development/debug image | 200 MB compressed |

These limits account for the Debian-slim base required by FairCom's deb packaging. Exceeding these limits requires written justification and Product approval. The build pipeline enforces these limits automatically.

---

## Base Image Requirements

FairCom produces deb and rpm packages only. Base image selection must be compatible with this packaging infrastructure.

**Approved base images:**

| Base Image | Use Case | Notes |
|------------|----------|-------|
| Debian-slim (preferred) | Standard deployments | Compatible with deb packages, ~30 MB base |
| Distroless (Debian-based) | Maximum minimization | No shell, no package manager; requires self-contained binaries |
| Scratch | Static binaries only | Smallest possible; requires fully static linking |

**Prohibited:**
- Ubuntu (unnecessarily large)
- Debian (full, not slim variant)
- CentOS/Rocky/RHEL (use rpm directly on host if needed)
- Alpine (requires apk packaging not currently produced)

**Future consideration:** If image size becomes a competitive requirement, Product may request Engineering evaluate apk package builds to enable Alpine-based images.

---

## Exclusion List

The following must be removed or never included:

### Documentation and Help
- Man pages (`/usr/share/man`)
- Info pages (`/usr/share/info`)
- Documentation directories (`/usr/share/doc`)
- README files
- Tutorial content
- Sample configurations (ship minimal defaults only)

### Binary Utilities
Remove all utilities not required for application execution:

- Shells beyond `/bin/sh` (no bash, zsh, fish)
- Text editors (vi, vim, nano, ed)
- Network diagnostic tools (ping, traceroute, netstat, ss, curl, wget)
- File utilities (find, locate, grep, awk, sed) unless operationally required
- Compression tools (gzip, tar, zip) unless required at runtime
- Package managers (apt, dpkg) in final image

### Development Artifacts
- Compilers and build tools
- Header files
- Static libraries (unless linked)
- Debug symbols
- Test suites

### System Components
- Cron
- Syslog (use stdout/stderr)
- SSH server
- Any init system beyond minimal process supervision

---

## Required Components

Include only:

1. **Application binaries** — Edge Suite executables
2. **Runtime dependencies** — shared libraries required for execution
3. **Minimal shell** — `/bin/sh` for entrypoint scripts (if needed)
4. **CA certificates** — for TLS validation
5. **Timezone data** — if application requires timezone handling
6. **Configuration defaults** — minimal, production-ready defaults

---

## Build Requirements

### Multi-stage builds
All images must use multi-stage Dockerfiles. Build dependencies stay in builder stages; only runtime artifacts copy to final image.

### Layer optimization
- Combine related operations in single RUN statements
- Order layers by change frequency (static first)
- Clean package manager caches in same layer as install
- Remove apt lists after install

### No secrets
Images must contain zero credentials, keys, or tokens. All secrets inject at runtime via environment variables or mounted volumes.

---

## Validation Checklist

Before release, verify:

- [ ] Image size within budget
- [ ] No prohibited utilities present
- [ ] No documentation or tutorials
- [ ] No package manager in final image
- [ ] Security scan completed (no critical/high vulnerabilities)
- [ ] Application starts and responds correctly
- [ ] Entrypoint handles signals properly (graceful shutdown)
- [ ] Docker Hub README reviewed and approved
- [ ] Tags follow naming convention
- [ ] Content trust signature valid

---

## Exception Process

Requests to include prohibited components or exceed limits require:

1. Written justification explaining operational necessity
2. Size impact assessment
3. Security review
4. Product approval

Exceptions are granted per-release and must be re-justified for each major version.

---

## Extension to Other Products

This specification serves as the template for containerizing additional FairCom products. When extending:

1. Create new repository under `faircom/` namespace
2. Apply same ownership and governance model
3. Adjust size budgets based on product footprint
4. Create product-specific README in docs repo
5. Configure pipelines following established patterns

Product-specific variations require documented justification and remain under Product ownership.

---

## Appendix A: Size Reduction Techniques

Reference for Engineering implementation:

```dockerfile
# Example: Debian-slim based minimal image
FROM debian:bookworm-slim AS builder
# ... build steps, install build dependencies ...

FROM debian:bookworm-slim

# Install runtime dependencies and clean up in single layer
RUN apt-get update \
    && apt-get install -y --no-install-recommends \
        ca-certificates \
        tzdata \
    && rm -rf /var/lib/apt/lists/* \
    && rm -rf /usr/share/man /usr/share/doc /usr/share/info

# Copy application from builder
COPY --from=builder /app/edge-suite /usr/local/bin/

# Run as non-root
USER nonroot

ENTRYPOINT ["/usr/local/bin/edge-suite"]
```

```dockerfile
# Example: Distroless for self-contained binaries
FROM gcr.io/distroless/base-debian12
COPY edge-suite /
ENTRYPOINT ["/edge-suite"]
```

---

## Appendix B: Pipeline Configuration Examples

### GitHub Actions (build and push)

```yaml
name: Docker Release - Edge Suite

on:
  push:
    tags:
      - 'v*.*.*'

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3
      
      - name: Login to Docker Hub
        uses: docker/login-action@v3
        with:
          username: ${{ secrets.DOCKERHUB_USERNAME }}
          password: ${{ secrets.DOCKERHUB_TOKEN }}
      
      - name: Extract version
        id: version
        run: echo "VERSION=${GITHUB_REF#refs/tags/v}" >> $GITHUB_OUTPUT
      
      - name: Build and push
        uses: docker/build-push-action@v5
        with:
          context: .
          push: true
          tags: |
            faircom/edge-suite:${{ steps.version.outputs.VERSION }}
            faircom/edge-suite:latest
      
      - name: Check image size
        run: |
          SIZE=$(docker image inspect faircom/edge-suite:${{ steps.version.outputs.VERSION }} --format='{{.Size}}')
          MAX_SIZE=78643200  # 75 MB in bytes
          if [ "$SIZE" -gt "$MAX_SIZE" ]; then
            echo "Image size ($SIZE bytes) exceeds limit ($MAX_SIZE bytes)"
            exit 1
          fi
      
      - name: Run security scan
        uses: aquasecurity/trivy-action@master
        with:
          image-ref: faircom/edge-suite:${{ steps.version.outputs.VERSION }}
          exit-code: 1
          severity: CRITICAL,HIGH
```

### README Sync Pipeline

```yaml
name: Sync Docker Hub README

on:
  push:
    branches: [main]
    paths:
      - 'edge-suite/README.md'

jobs:
  sync:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      - name: Push README to Docker Hub
        uses: peter-evans/dockerhub-description@v3
        with:
          username: ${{ secrets.DOCKERHUB_USERNAME }}
          password: ${{ secrets.DOCKERHUB_TOKEN }}
          repository: faircom/edge-suite
          readme-filepath: ./edge-suite/README.md
```

---

## Change Log

| Date | Author | Change |
|------|--------|--------|
| 2026-03 | [Name] | Initial draft |
| 2026-03 | [Name] | Added Docker Hub, documentation, pipeline requirements |
| 2026-03 | [Name] | Scoped to Edge Suite; adjusted base images for deb packaging |