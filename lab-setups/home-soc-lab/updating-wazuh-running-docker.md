# Updating Wazuh Running Docker

**Tags:** `wazuh` `docker` `siem` `homelab` `soc`

### Introduction

Running Wazuh in Docker is great for homelabs and SOC setups — but keeping it updated requires a few careful steps. In this post I'll walk through how I upgraded my Wazuh single-node Docker deployment, including the mistakes I made along the way and how to recover from them.

### Prerequisites

Here are some prerequites;

* Wazuh deployed via Docker Compose (single-node or multi-node)
* The `wazuh-docker` Git repository cloned locally
* Docker and Docker Compose installed

### Step 1: Bring the container down and fetch the Latest Tags

Navigate to your `wazuh-docker` directory and bring it down:

```bash
cd /path/to/wazuh-docker
docker compose -f single-node/docker-compose.yml down
```

&#x20;Fetch all remote tags:

```bash
git fetch --all --tags --force
```

The `--force` flag is important here. Without it, conflicting local tags (e.g. a beta tags) willl be rejected and not all tags may pull cleanly.

Here was my output

```
f584612e..4161af02  4.14.5  -> origin/4.14.5  #The 4.14.5 branch got new commits upstream
90f233ea..98505408  4.14.6  -> origin/4.14.6
b976edbb..21e4202e  main    -> origin/main
* [new branch]      merge-4.14.6-into-main -> origin/merge-4.14.6-into-main   ##A brand new branch appeared on the remote
* [new tag]         v4.14.5-rc1 -> v4.14.5-rc1    ##A release candidate tag was fetched (pre-release, not stable)
! [rejected]        v5.0.0-beta1 -> v5.0.0-beta1  (would clobber existing tag)    ##A local tag conflicts with the remote 
```

> 💡 **RC vs Stable:** A tag like `v4.14.5-rc1` is a **Release Candidate** — a pre-release test version. The stable release is simply `v4.14.5` (no `-rc` suffix).

To verify which stable tags are available, run the following

```bash
git tag | grep -v rc | grep -v beta | sort -V
```

### Step 2: Check for Local Changes

Before switching versions, we check if there are any local modifications:

```bash
git diff single-node/config/wazuh_dashboard/wazuh.yml
```

In my case, I had a single line added from when I enrolled an agent:

```diff
+enrollment.dns: "192.168.100.3"
```

Make a note of any custom changes — you'll need to re-apply them after the upgrade.

### Step 3: Handle Local Changes

Since I knew exactly what my change was (just the `enrollment.dns` line), I discarded it to allow a clean checkout:

```bash
git checkout -- single-node/config/wazuh_dashboard/wazuh.yml
```

Alternatively, if there were more complex changes, I would have stashed them to preserve them by running:

```bash
git stash
# ... do your checkout and upgrade ...
git stash pop   # restore changes afterwards
```

### Step 4: Checkout the Target Version

I wanted to roll with the most stable version at the time of writing this and so, I checked out version 4.14.4

```bash
git checkout v4.14.4
```

Successful output looks like:

```
Previous HEAD position was 0f15acb7 Merge pull request #1789 ...
HEAD is now at 7af31ddf Merge pull request #2245 ...
```

### Step 5: Bring Up the New Version and Verify the upgrade

After the upgrade, we need to bring the container up by running:

```bash
docker compose -f /docker-compose.yml up -d
```

Finally to verify the upgrade was successful and the relevant containers are running:

```bash
docker ps
```

You should see `wazuh.manager`, `wazuh.indexer`, and `wazuh.dashboard` all on the new version tag.

If any startup errors, monitor them at:

```bash
docker compose -f /docker-compose.yml logs -f
```

### What About Agents?

Agents are **not** upgraded automatically when you upgrade the central components. We'll need to upgrade them manually or via the Wazuh API. You can do this from the dashboard under **Agent Management → Summary**, or using the `agent_upgrade` CLI tool.

### Recovering from a Skipped `down`

When doing this, I skipped the first step of first bringing down the container. But not to worry. Here is how I effected the changes.

A simple `restart` won't fully fix it since the containers may still be running the old image. A proper down/up cycle puts things inorder.

```bash
docker compose -f single-node/docker-compose.yml down
docker compose -f single-node/docker-compose.yml up -d
```

This forces Docker to fully recreate the containers from the new images.

***

### Summary

| Step                     | Command                                                  |
| ------------------------ | -------------------------------------------------------- |
| Fetch latest tags        | `git fetch --all --tags --force`                         |
| Check local changes      | `git diff <file>`                                        |
| Discard or stash changes | `git checkout -- <file>` or `git stash`                  |
| Checkout target version  | `git checkout v4.14.4`                                   |
| Stop old containers      | `docker compose -f single-node/docker-compose.yml down`  |
| Start new version        | `docker compose -f single-node/docker-compose.yml up -d` |
| Verify                   | `docker ps` + `docker compose logs -f`                   |

***

_Happy hunting 🛡️_
