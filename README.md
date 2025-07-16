# qctl - Qwilt CLI

The qctl is the main command-line interface for interacting with Qwilt Cloud services.

## Overview

qctl provides a unified interface for managing Qwilt Cloud resources including
authentication, clusters, applications, images, and more.

## Installation

For macOS users, use Homebrew to install qctl:

```bash
brew install qwilt/tap/qctl
```

For Linux and Windows users, download the latest release from the [GitHub
releases page](https://github.com/Qwilt/qctl/releases).

## Usage

```bash
qctl [command] [flags]
```

## Available Commands

### Authentication Commands (`qctl auth`)

Manage authentication and authorization.

- **`qctl auth login`** - Login to Qwilt Cloud
  - Flags:
    - `-o, --org-id string` - Organization ID (optional)
    - `-u, --username string` - Username (optional, prompted if not provided)
    - `-p, --password string` - Password (optional, prompted if not provided)
  - Examples:
    ```bash
    qctl auth login
    qctl auth login --org-id my-org-id
    qctl auth login -o my-org-id -u myuser
    ```

- **`qctl auth print-token`** - Print the access token from local cache
  - Displays the current access token in ExecCredential format for kubectl integration

### Cluster Commands (`qctl clusters`)

Manage clusters in the system.

- **`qctl clusters list`** - List all clusters
- **`qctl clusters get [cluster-name]`** - Get cluster details
- **`qctl clusters create`** - Create a new cluster
- **`qctl clusters update`** - Update an existing cluster
- **`qctl clusters delete [cluster-name...]`** - Delete one or more clusters
  - Supports multiple deletion methods:
    - By name: `qctl clusters delete cluster1 cluster2`
    - By selector: `qctl clusters delete -l environment=test`
    - All resources: `qctl clusters delete --all`
    - From file: `qctl clusters delete -f cluster-definition.yaml`
  - Common flags: `--force`, `--grace-period`, `--timeout`, `--dry-run`, `--cascade`
- **`qctl clusters explain`** - Explain cluster resource format
- **`qctl clusters connect [cluster-name]`** - Connect to a cluster
  - Adds cluster configuration to your kubeconfig for kubectl access

### Application Commands (`qctl applications`)

Manage applications in the system. Can also be used with aliases: `apps`, `app`,
`application`.

- **`qctl applications list`** - List all applications
- **`qctl applications get [app-name]`** - Get application details
- **`qctl applications create`** - Create a new application
- **`qctl applications update`** - Update an existing application
- **`qctl applications delete [app-name...]`** - Delete one or more applications
  - Supports multiple deletion methods:
    - By name: `qctl applications delete app1 app2`
    - By selector: `qctl applications delete -l version=old`
    - All resources: `qctl applications delete --all`
    - From file: `qctl applications delete -f app-definition.yaml`
  - Common flags: `--force`, `--grace-period`, `--timeout`, `--dry-run`, `--cascade`
- **`qctl applications explain`** - Explain application resource format

### Image Commands (`qctl images`)

Manage images in the system.

- **`qctl images list`** - List all images
- **`qctl images get [image-name]`** - Get image details
- **`qctl images create`** - Create a new image
- **`qctl images update`** - Update an existing image
- **`qctl images delete [image-name...]`** - Delete one or more images
  - Supports multiple deletion methods:
    - By name: `qctl images delete image1 image2`
    - By selector: `qctl images delete -l arch=amd64`
    - All resources: `qctl images delete --all`
    - From file: `qctl images delete -f image-definition.yaml`
  - Common flags: `--force`, `--grace-period`, `--timeout`, `--dry-run`, `--cascade`
- **`qctl images explain`** - Explain image resource format
- **`qctl images push [path-to-qcow2-file]`** - Push a qcow2 image to Qwilt's image registry
  - Flags:
    - `-t, --tag string` - Tag for the image (default: "latest")
  - Examples:
    ```bash
    qctl images push my-image.qcow2
    qctl images push my-image.qcow2 --tag v1.0
    qctl images push /path/to/my-image.qcow2 --tag latest
    ```

### Version Command

- **`qctl version`** - Display version information

## Zone Proxy Mode

qctl supports a special zone proxy mode that allows you to run kubectl commands
directly against zone-specific Kubernetes API servers.

### Zone Proxy Usage

When the `--zone` flag is specified, qctl will:

1. Fetch the zone details to get the API endpoint
2. Authenticate using your existing qctl token
3. Execute kubectl commands against the zone's API server (with `--insecure-skip-tls-verify`)

```bash
# List pods in a specific zone
qctl --zone sagim1 get pods

# Get deployments in a zone with custom output
qctl --zone sagim1 get deployments -o yaml

# Apply a manifest to a specific zone
qctl --zone sagim1 apply -f deployment.yaml

# Any kubectl command works
qctl --zone sagim1 logs deployment/my-app
```

### How Zone Proxy Works

1. **Zone Discovery**: `qctl zones get <zone-name>` is executed to fetch zone details
2. **Endpoint Extraction**: The `status.apiEndpoint` field is extracted from the zone resource
3. **Authentication**: The same token from `qctl auth print-token` is used
4. **Kubectl Execution**: `kubectl --server <endpoint> --token <token> --insecure-skip-tls-verify=true <args>` is executed

### Implementation Details

qctl now uses direct kubectl flags instead of kubeconfig files for **all operations**:

- **Regular qctl commands** (e.g., `qctl zones list`) use `kubectl --server <global-mgmt-url> --token <token> --insecure-skip-tls-verify=true`
- **Zone proxy commands** (e.g., `qctl --zone sagim1 get pods`) use `kubectl --server <zone-url> --token <token> --insecure-skip-tls-verify=true`

This approach eliminates the need to maintain kubeconfig files under `~/.qc/kubeconfig` and provides a cleaner, more direct interface to Kubernetes APIs.

### Error Handling

If the specified zone doesn't exist, qctl will show an error message along with
a list of available zones:

```bash
$ qctl --zone nonexistent-zone get pods
Fetching API endpoint for zone 'nonexistent-zone'...
Error: Zone 'nonexistent-zone' not found.

Available zones:
  - zone1
  - zone2
  - production-east
  - staging-west
```

### Prerequisites

- You must be authenticated with `qctl auth login`
- The target zone must exist and have a valid `status.apiEndpoint`
- `kubectl` must be installed and available in your PATH

## Global Flags

Most commands support standard flags for output formatting and configuration.

- `--zone string` - Target zone to run kubectl commands against (enables zone proxy mode)
- `--verbose` - Enable verbose output with detailed progress information

### Verbose Mode

Use the `--verbose` flag to get detailed information about qctl operations:

```bash
# Normal mode - clean, minimal output
qctl --zone sagim1 get pods

# Verbose mode - detailed progress and debug information
qctl --zone sagim1 get pods --verbose
```

In verbose mode, you'll see:

- Detailed connection information including API endpoints
- Authentication progress
- Full kubectl commands being executed
- Additional debug information

## Configuration

qctl stores configuration in the user's home directory. Configuration includes:

- **Username**: Authenticated user
- **Organization ID**: Default organization
- **Tokens**: Cached authentication tokens

## Authentication

qctl uses token-based authentication. After logging in with `qctl auth login`,
tokens are cached locally and used for subsequent API calls. The `qctl auth
print-token` command can be used for kubectl integration.

### Authentication Methods

1. **Interactive Login**: Use `qctl auth login` to authenticate with username and password
2. **Environment Variables**: Set `QC_USERNAME` and `QC_PASSWORD` for automatic authentication
3. **API Token**: Set `QC_TOKEN` environment variable for direct API access (no login required)

### Examples

```bash
# Method 1: Interactive login
qctl auth login

# Method 2: Environment variables for automatic login
export QC_USERNAME=user@example.com
export QC_PASSWORD=your_password
qctl applications list

# Method 3: API token (no login needed)
export QC_TOKEN=your_api_token
qctl applications list
```

### Getting Started

1. Login to Qwilt Cloud:
   ```bash
   qctl auth login
   ```

2. List available clusters:
   ```bash
   qctl clusters list
   ```

3. Connect to a cluster:
   ```bash
   qctl clusters connect my-cluster
   ```

### Managing Images

1. Push a qcow2 image:
   ```bash
   qctl images push my-vm-image.qcow2 --tag v1.0
   ```

2. List available images:
   ```bash
   qctl images list
   ```

## Help

For help with any command, use the `--help` flag:

```bash
qctl --help
qctl auth --help
qctl clusters connect --help
qctl applications --help
qctl applications delete --help
```

## Troubleshooting

- If authentication fails, ensure you're using the correct credentials and organization ID
- Use verbose logging with `-v` flag for debugging
