# Akuity Kargo Quickstart — Daniel Lifchitz

## Setup Overview 
This repo implements the Akuity Kargo quickstart tutorial, deploying a three-stage GitOps promotion pipeline (dev → staging → prod) for the guestbook application, with Argo CD managing cluster reconciliation and Kargo managing environment promotion.

**Environment:**
- Kubernetes: k3d cluster via GitHub Codespaces
- Image registry: GitHub Container Registry (ghcr.io)
- GitOps repo: this repo (public)
- Kargo instance: hosted on Akuity Platform
- Argo CD instance: hosted on Akuity Platform

## Architecture (OOB)
Warehouse (image subscription: ghcr.io/danlif20/guestbook, SemVer)   
 ↓
[dev stage] → manual promotion of new freight, kustomize write-back to app/env/dev   
 ↓ (verified)
[staging stage] → manual promotion, kustomize write-back to app/env/staging  
  ↓ (verified)
[prod stage] → manual promotion, PR-based write-back (not direct commit)

Argo CD applications are linked to each stage via argoCDAppUpdates,so Kargo triggers a sync after each successful promotion.

## Key Design Decisions

**PR-based promotion for prod:**  

Rather than committing directly to main for prod, Kargo opens a pull request. This enforces a human review step, provides a clear audit trail in GitHub, and makes rollback straightforward (revert the PR). This is a compliance-friendly pattern for regulated environments.

**SemVer tag filtering on the Warehouse:**  
The Warehouse uses tagSelectionStrategy: SemVer to ignore non-versioned tags (e.g., latest, main). This prevents accidental promotion of unversioned images and enforces a clean release discipline.

**Kustomize for image updates:** 
 Each stage uses kustomize edit set image to write the promoted image tag back to the repo. This keeps the GitOps contract intact,the repo is always the source of truth for what's running, and changes are auditable as Git commits.





**Agent-based Argo CD integration:** 

 The Kargo-to-Argo CD link uses Akuity's managed agent model. The agent in each cluster calls outbound to the Akuity control plane,no inbound firewall rules needed, and cluster credentials stay cluster-side. This is architecturally cleaner than self-managed Argo CD in an enterprise context.

## Assumptions

- A single cluster runs all three stages (namespaced isolation). In production, each stage would typically map to a separate cluster.
- The guestbook image is used as a stand-in for a real application. The promotion mechanics are identical regardless of application complexity.
- GitHub PAT credentials are scoped to this repo and expire in 7 days per the tutorial guidance.

## Enhancement: 

Auto-Promotion on dev: Added “autoPromotionPolicies” to the dev stage in “kargo/project.yaml”. New freight now promotes to dev automatically when detected in the guestbook, even though this would not be ideal in a real enterprise environment. For the purposes of this exercise it highlights the cascading through the entire lifecycle without requiring the user to manually click "Promote" at every gate.

## What Surprised Me

The PR-based promotion for prod is more powerful than it looks. Kargo doesn't just open a PR, it writes the exact diff needed, targets the right branch, and then waits for the merge before marking the stage healthy. That gives you GitHub's native review workflow as a promotion gate without any custom scripting.

## What Felt Complex

The mental model of where Kargo ends and Argo CD begins took a moment to internalize. Kargo writes the promotion back to Git and triggers the sync , however, Argo CD is still the authority on whether the deployment actually succeeded. Until you see it working end-to-end, it's not immediately obvious that you're dealing with two reconciliation loops running in sequence rather than one continuous pipeline. Once it clicked, the design made complete sense but, it's a concept worth front-loading when onboarding new users to the platform. 

## Favorite Feature

Auto-promotion + Kargo Warehouse SemVer filtering together. The combination means a developer pushes a semver-tagged image, and within minutes it's running in dev without anyone touching the CD system. That's the GitOps promise actually working end-to-end.


