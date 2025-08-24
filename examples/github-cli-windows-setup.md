# GitHub CLI on Windows - Setup and Usage

## Problem
GitHub CLI (`gh`) commands not working through bash/WSL on Windows, causing PR creation to fail.

## Solution

### Installation
1. Download GitHub CLI from: https://cli.github.com/
2. Install using Windows installer (.msi file)
3. **Restart terminal/command prompt** after installation

### Usage on Windows

#### Option 1: Use PowerShell directly
```powershell
# Open PowerShell (not through WSL/bash)
gh --version
gh pr create --title "Your Title" --body "Your description"
```

#### Option 2: Use Command Prompt
```cmd
# Open Command Prompt (cmd.exe)
gh --version
gh pr create --title "Your Title" --body "Your description"
```

#### Option 3: From WSL/bash (if configured)
```bash
# May require PATH updates or using full path
/mnt/c/Program\ Files/GitHub\ CLI/gh.exe --version
```

### Authentication
```bash
gh auth login
# Follow prompts to authenticate with GitHub
```

## Key Gotchas
- **Path Issues**: GitHub CLI may not be in PATH for bash/WSL sessions
- **Terminal Restart**: Always restart terminal after installation
- **Authentication**: Must authenticate before creating PRs
- **Repository Context**: Must be in git repository directory

## Alternative: Manual PR Creation
If GitHub CLI unavailable, create PRs through GitHub web interface:
1. Go to repository on GitHub.com
2. Click "Pull requests" tab
3. Click "New pull request" 
4. Select branch and create PR manually

## Date Added
2025-08-24

## Context
Encountered during `/generate-prp INITIAL.md` command execution when trying to create automated pull request for project setup.