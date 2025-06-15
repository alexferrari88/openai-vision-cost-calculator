# GitHub Actions Publishing Setup

This repository includes GitHub Actions workflows for automated testing and publishing to PyPI. This guide explains how to set up and use these workflows.

## Workflows Overview

### 1. CI Workflow (`ci.yml`)
- **Triggers**: Push to `master`/`main` branches, Pull Requests
- **Purpose**: Run tests and code quality checks
- **Matrix**: Tests on Python 3.8-3.12
- **Checks**: Black, isort, flake8, mypy, pytest
- **Coverage**: Uploads to Codecov

### 2. Publish Workflow (`publish.yml`)
- **Triggers**: Published releases
- **Purpose**: Automatically publish to PyPI/Test PyPI
- **Flow**: Test → Build → Publish → Notify
- **Smart Publishing**: 
  - Pre-releases → Test PyPI
  - Official releases → Production PyPI

## Required GitHub Secrets

You need to set up the following secrets in your GitHub repository:

### 1. PyPI API Tokens

Go to your repository → Settings → Secrets and variables → Actions

Create the following secrets:

#### `PYPI_API_TOKEN`
1. Go to [PyPI Account Settings](https://pypi.org/manage/account/)
2. Navigate to "API tokens"
3. Click "Add API token"
4. Name: `github-actions-openai-vision-cost`
5. Scope: "Entire account" (or specific to this project once published)
6. Copy the token (starts with `pypi-`)
7. Add as GitHub secret: `PYPI_API_TOKEN`

#### `TEST_PYPI_API_TOKEN`
1. Go to [Test PyPI Account Settings](https://test.pypi.org/manage/account/)
2. Navigate to "API tokens"
3. Click "Add API token"
4. Name: `github-actions-openai-vision-cost-test`
5. Scope: "Entire account"
6. Copy the token (starts with `pypi-`)
7. Add as GitHub secret: `TEST_PYPI_API_TOKEN`

### 2. Optional: Codecov Token

For code coverage reporting:

#### `CODECOV_TOKEN`
1. Go to [Codecov](https://codecov.io/)
2. Sign in with GitHub and add this repository
3. Copy the repository token
4. Add as GitHub secret: `CODECOV_TOKEN`

## Environment Protection (Recommended)

Set up environment protection to add manual approval for production releases:

1. Go to repository → Settings → Environments
2. Create two environments:
   - `test-pypi` (no protection needed)
   - `pypi` (add protection rules)

### For `pypi` environment:
- **Required reviewers**: Add yourself or team members
- **Wait timer**: Optional (e.g., 5 minutes)
- **Deployment branches**: Only `master`/`main`

## How to Use

### Automatic Publishing on Release

1. **Create a Release**:
   ```bash
   # Tag your release
   git tag v1.0.1
   git push origin v1.0.1
   ```

2. **GitHub Release**:
   - Go to repository → Releases → "Create a new release"
   - Choose your tag (v1.0.1)
   - Fill in release notes
   - **For testing**: Check "Set as pre-release"
   - **For production**: Leave "Set as pre-release" unchecked
   - Click "Publish release"

3. **Automatic Process**:
   - Pre-release → Publishes to Test PyPI
   - Official release → Publishes to Production PyPI
   - Both run full test suite first

### Manual Testing

You can also trigger workflows manually:

```bash
# Run CI manually
gh workflow run ci.yml

# Check workflow status
gh run list
```

## Version Management

The package version is managed in `src/openai_vision_cost/__init__.py`:

```python
__version__ = "1.0.1"
```

**Important**: Always update the version before creating a release:

1. Update `__version__` in `__init__.py`
2. Commit the version bump
3. Create and push the tag
4. Create the GitHub release

## Troubleshooting

### Common Issues

#### 1. `400 Client Error: File already exists`
- The version already exists on PyPI
- Update the version number in `__init__.py`

#### 2. `403 Client Error: Invalid or non-existent authentication information`
- Check that API tokens are correctly set in GitHub secrets
- Ensure tokens haven't expired
- Verify token permissions (should be "Entire account" or project-specific)

#### 3. Tests failing
- All tests must pass before publishing
- Check the CI workflow logs for specific errors
- Fix issues locally and push fixes

#### 4. Environment protection blocking
- If you set up environment protection, you need to manually approve deployments
- Go to repository → Actions → find the workflow run → Review deployments

### Debugging

1. **Check workflow logs**:
   - Go to repository → Actions
   - Click on the failed workflow run
   - Expand the failing step to see detailed logs

2. **Test locally**:
   ```bash
   # Run the same commands locally
   pytest
   black --check src tests
   python -m build
   twine check dist/*
   ```

3. **Test PyPI first**:
   - Always test with pre-releases first
   - Verify the package installs correctly from Test PyPI:
   ```bash
   pip install -i https://test.pypi.org/simple/ openai-vision-cost
   ```

## Security Best Practices

1. **Never commit secrets** to the repository
2. **Use environment protection** for production publishing
3. **Regularly rotate API tokens** (every 6-12 months)
4. **Monitor workflow runs** for suspicious activity
5. **Use specific token scopes** when possible

## Workflow Customization

### Modify Python versions
In both workflows, update the matrix:

```yaml
strategy:
  matrix:
    python-version: ["3.8", "3.9", "3.10", "3.11", "3.12"]
```

### Add additional checks
Add steps to the CI workflow:

```yaml
- name: Security check
  run: pip install safety && safety check
```

### Modify notification
Customize the notification step in `publish.yml`:

```yaml
- name: Slack notification
  uses: 8398a7/action-slack@v3
  with:
    status: ${{ job.status }}
    webhook_url: ${{ secrets.SLACK_WEBHOOK }}
```

## Advanced Features

### Trusted Publishing (Alternative to API Tokens)

PyPI supports "Trusted Publishing" which eliminates the need for API tokens:

1. Configure in PyPI project settings
2. Modify workflow to use OIDC token
3. Remove API token secrets

See [PyPI Trusted Publishing Guide](https://docs.pypi.org/trusted-publishers/) for details.

### Multi-environment Publishing

For complex scenarios, you can add staging environments:

```yaml
publish-staging:
  needs: [test, build]
  runs-on: ubuntu-latest
  if: github.ref == 'refs/heads/develop'
  environment: staging
  # ... steps to publish to staging PyPI
```