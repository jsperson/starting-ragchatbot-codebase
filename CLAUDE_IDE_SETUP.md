Fixing Claude Code IDE Integration in VS Code Remote Containers

Problem
When using VS Code's "Attach to Running Container" feature, Claude Code's IDE integration (diff views, selection context sharing, etc.) doesn't work because VS Code connects as root while you're running Claude Code as a different user (e.g., developer). The user mismatch prevents the IPC socket communication between Claude Code and VS Code.

Solution
When attaching to a running container (rather than using Dev Containers to manage the container lifecycle), VS Code ignores devcontainer.json. You need to configure the remote user in the attached container configuration file instead:

Open Command Palette (Cmd+Shift+P / Ctrl+Shift+P)
Run "Dev Containers: Open Attached Container Configuration File..."
Select your container image
Add remoteUser to the config:

json{
    "remoteUser": "developer",
    "workspaceFolder": "/home/developer/source/your-project",
    "extensions": [
        "anthropic.claude-code"
    ]
}

Save the file
Disconnect from the container (click bottom-left status bar → "Close Remote Connection")
Reattach to the container
Verify with whoami in the integrated terminal

Once both VS Code and your terminal session are running as the same user, Claude Code's IDE integration should work correctly.

Explanation
This issue occurs because:
- VS Code's "Attach to Running Container" connects as the container's default user (often root)
- Claude Code CLI runs as a different user (e.g., developer) for security/permissions
- IDE integration requires both to run as the same user to share IPC socket communication
- Unlike Dev Containers, attached containers ignore devcontainer.json configuration

The solution forces VS Code to connect as the developer user by configuring the attached container settings directly. This ensures both VS Code and Claude Code run as the same user, enabling proper IPC communication for IDE features like diff views, selection sharing, and keyboard shortcuts.
