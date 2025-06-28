# AirPilot Agents

## Available Agents

| Agent                           | Model                                                                               | Permissions                                              | Description                                                                                                                                                                                                                                                                                                                                   | Default Task                               |
| ------------------------------- | ----------------------------------------------------------------------------------- | -------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------ |
| `Git Commit Bot`<br/>`bot`      | <img src="https://img.shields.io/badge/Sonnet-blue?style=flat-square" alt="Sonnet"> | ✔ Read<br/>✔ Write<br/>✔ Network<br/>✘ Sandbox | **Automate your Git workflow with intelligent commit messages**<br/><br/>Analyzes Git repository changes, generates detailed commit messages following Conventional Commits specification, and pushes changes to remote repository.                                                                                                           | "Push all changes."                        |
| `Security Scanner`<br/>`shield` | <img src="https://img.shields.io/badge/Opus-purple?style=flat-square" alt="Opus">   | ✔ Read<br/>✔ Write<br/>✘ Network<br/>✘ Sandbox | **Advanced AI-powered Static Application Security Testing (SAST)**<br/><br/>Performs comprehensive security audits by spawning specialized sub-agents for: codebase intelligence gathering, threat modeling (STRIDE), vulnerability scanning (OWASP Top 10, CWE), exploit validation, remediation design, and professional report generation. | "Review the codebase for security issues." |
| `Unit Tests Bot`<br/>`code`     | <img src="https://img.shields.io/badge/Opus-purple?style=flat-square" alt="Opus">   | ✔ Read<br/>✔ Write<br/>✘ Network<br/>✘ Sandbox | **Automated comprehensive unit test generation for any codebase**<br/><br/>Analyzes codebase and generates comprehensive unit tests by: analyzing code structure, creating test plans, writing tests matching your style, verifying execution, optimizing coverage (>80% overall, 100% critical paths), and generating documentation.         | "Generate unit tests for this codebase."   |

### Available Icons

Choose from these icon options when creating agents:

- `bot` - General purpose
- `shield` - Security related
- `code` - Development
- `terminal` - System/CLI
- `database` - Data operations
- `globe` - Network/Web
- `file-text` - Documentation
- `git-branch` - Version control

---

## Importing Agents

### Method 1: Import from GitHub (Recommended)

1. In AirPilot, navigate to **Agents**
2. Click the **Import** dropdown button
3. Select **From GitHub**
4. Browse available agents from the official repository
5. Preview agent details and click **Import Agent**

### Method 2: Import from Local File

1. Download a `.airpilot.json` file from this repository
2. In AirPilot, navigate to **Agents**
3. Click the **Import** dropdown button
4. Select **From File**
5. Choose the downloaded `.airpilot.json` file

## Exporting Agents

### Export Your Custom Agents

1. In AirPilot, navigate to **Agents**
2. Find your agent in the grid
3. Click the **Export** button
4. Choose where to save the `.airpilot.json` file

### Agent File Format

All agents are stored in `.airpilot.json` format with the following structure:

```json
{
  "version": 1,
  "exported_at": "2025-01-23T14:29:58.156063+00:00",
  "agent": {
    "name": "Your Agent Name",
    "icon": "bot",
    "model": "opus|sonnet|haiku",
    "system_prompt": "Your agent's instructions...",
    "default_task": "Default task description",
    "sandbox_enabled": false,
    "enable_file_read": true,
    "enable_file_write": true,
    "enable_network": false
  }
}
```

## Technical Implementation

### How Import/Export Works

The agent import/export system is built on a robust architecture:

#### Backend (Rust/Tauri)

- **Storage**: SQLite database stores agent configurations
- **export**: Serializes agent data to JSON with version control
- **Import**: Validates and deduplicates agents on import
- **GitHub Integration**: Fetches agents via GitHub API

#### Frontend (React/TypeScript)

- **UI Components**:
  - `Agents.tsx` - Main agent management interface
  - `GitHubAgentBrowser.tsx` - GitHub repository browser
  - `CreateAgent.tsx` - Agent creation/editing form
- **File Operations**: Native file dialogs for import/export
- **Real-time Updates**: Live agent status and execution monitoring

### Key Features

1. **Version Control**: Each agent export includes version metadata
2. **Duplicate Prevention**: Automatic naming conflict resolution
3. **Permission System**: Granular control over file, network, and sandbox access
4. **Model Selection**: Choose between Opus, Sonnet, and Haiku models
5. **GitHub Integration**: Direct import from the official repository

## Contributing

We welcome agent contributions! Here's how to add your agent:

### 1. Create Your Agent

Design and test your agent in AirPilot with a clear, focused purpose.

### 2. export Your Agent

export your agent to a `.airpilot.json` file with a descriptive name.

### 3. Submit a Pull Request

1. Fork this repository
2. Add your `.airpilot.json` file to the `agents` directory
3. Update this README with your agent's details
4. Submit a PR with a description of what your agent does

### Agent Guidelines

- **Single Purpose**: Each agent should excel at one specific task
- **Clear Documentation**: Write comprehensive system prompts
- **Safe Defaults**: Be conservative with permissions
- **Model Choice**: Use Haiku for simple tasks, Sonnet for general purpose, Opus for complex reasoning
- **Naming**: Use descriptive names that clearly indicate the agent's function

## License

These agents are provided under the same license as the AirPilot project. See the main LICENSE file for details.
