# Docker Build Reusable Workflows (Removed)

**Status: Removed — this approach was attempted but did not work.**

This log is kept for reference so the same approach isn't retried without knowing it was already tried and failed.

## What Was Tried

Reusable workflows to build container images with Buildah/Skopeo and push to Azure Container Registry (ACR) or AWS Elastic Container Registry (ECR), using OIDC-based auth (no long-lived passwords):

- `.github/workflows/docker-build-acr.yml`
- `.github/workflows/docker-build-ecr.yml`
- Example callers: `examples/docker-build-acr.yml`, `examples/docker-build-ecr.yml`

## Outcome

This solution did not work and has been removed from the repository. The workflow and example files were deleted; this document remains as a record of the attempt.
