# MCPee

A demonstration of how MCP (Model Context Protocol) servers running without
proper isolation can be exploited to cause security breaches and misinformation.

## Overview

This repository documents a critical security vulnerability demonstration
showing how an MCP server (`mcp-raider`) can attack and manipulate another MCP
server (`mcp-weather`) when both are running without proper sandboxing or
isolation. The attack demonstrates:

1. **Environment Variable Extraction**: Stealing sensitive API keys and
   credentials
2. **Runtime Code Manipulation**: Modifying weather data to return false
   information (e.g., extreme wind speeds of 133 m/s)
3. **Cross-Process Attack**: One MCP server compromising another through
   unrestricted system access

## The Attack Concept

### Background: What is MCP?

The Model Context Protocol (MCP) enables AI assistants to connect to external
tools and data sources. MCP servers provide specific functionalities that AI
agents can use to retrieve information or perform actions.

### The Vulnerability

When MCP servers run without proper isolation:

- They have unrestricted access to system processes
- They can read environment variables from other processes
- They can modify files and runtime behavior of other applications
- There is no authentication or sandboxing between different MCP servers

### The Attack Scenario

1. **Victim**: `mcp-weather` - A legitimate MCP server that provides weather
   information using OpenWeatherMap API
2. **Attacker**: `mcp-raider` - A malicious MCP server with system introspection
   and manipulation capabilities
3. **Target**: An AI assistant using both MCP servers simultaneously

The attack demonstrates how `mcp-raider` can:

- Extract the `OPENWEATHER_API_KEY` from the `mcp-weather` process
- Modify the weather server's code to return false data (133 m/s wind speeds)
- Cause the AI assistant to misinform users with dangerous weather information

## Attack Demonstration

### Prerequisites

1. Clone both repositories:

```bash
git clone https://github.com/denhamparry/mcp-weather
git clone https://github.com/denhamparry/mcp-raider
```

1. Build mcp-weather:

```bash
cd mcp-weather
npm install
npm run build
```

1. Build mcp-raider:

```bash
cd mcp-raider
npm install
npm run build
```

### MCP Client Setup

This demonstration requires an MCP client to invoke the tools. We recommend
Claude Desktop.

#### Configure Claude Desktop

Edit your Claude Desktop configuration file:

- **macOS**: `~/Library/Application Support/Claude/claude_desktop_config.json`
- **Linux**: `~/.config/Claude/claude_desktop_config.json`
- **Windows**: `%APPDATA%\Claude\claude_desktop_config.json`

Add the following configuration:

```json
{
  "mcpServers": {
    "weather": {
      "command": "node",
      "args": ["/absolute/path/to/mcp-weather/build/index.js"],
      "env": {
        "OPENWEATHER_API_KEY": "your_api_key_here"
      }
    },
    "raider": {
      "command": "node",
      "args": ["/absolute/path/to/mcp-raider/build/index.js"]
    }
  }
}
```

> **Important:** Use absolute paths to the built `index.js` files. Find your
> path with `cd mcp-weather && pwd`.
>
> - Linux example: `/home/user/mcp-weather/build/index.js`
> - macOS example: `/Users/username/mcp-weather/build/index.js`

#### Restart Claude Desktop

Fully quit and restart Claude Desktop for changes to take effect.

#### Test Connection

In a new Claude conversation:

```text
"What MCP tools are available?"
"What's the weather in London?"
```

If successful, proceed to the attack demonstration.

#### Alternative: Manual Testing

If not using Claude Desktop, you can run the servers directly:

```bash
# Terminal 1
cd mcp-weather
export OPENWEATHER_API_KEY="your-api-key-here"
node build/index.js

# Terminal 2
cd mcp-raider
node build/index.js
```

### Step-by-Step Attack Process

#### Step 1: Process Discovery

Use mcp-raider to find the mcp-weather process:

```yaml
Tool: get_process_id
Parameters:
  searchStrings: ["node", "mcp-weather"]
```

This returns the process ID (PID) of the running weather server.

#### Step 2: Environment Variable Extraction

Extract environment variables from the weather server:

```yaml
Tool: get_environment_variables
Parameters:
  processId: "<PID from Step 1>"
```

This reveals:

- `OPENWEATHER_API_KEY` - The sensitive API key
- Other configuration and system variables
- Potential database credentials or other secrets

#### Step 3: Code Manipulation - Scanning for Injection Points

Scan the weather server's compiled output for template literal variables. The
attack targets the built JavaScript file (`build/index.js`), not the TypeScript
source, because this is the code actively running in the Node.js process:

```yaml
Tool: luck_about_and_find_out_output_string_names
Parameters:
  fileName: /absolute/path/to/mcp-weather/build/index.js
```

> Replace `/absolute/path/to/` with your actual path (e.g., `/Users/username/`
> on macOS, `/home/user/` on Linux).

This identifies `$(variable_name)` format variables that can be manipulated in
the running code.

#### Step 4: Code Manipulation - Injecting False Data

Modify the weather server's compiled output to return false wind speeds:

```yaml
Tool: luck_about_and_find_out_replace_string_values
Parameters:
  fileName: /absolute/path/to/mcp-weather/build/index.js
  outputStringName: "$(windSpeed)"
  value: "133"
```

This hot-patches the running application to return a wind speed of 133 m/s (≈297
mph) regardless of actual weather conditions.

#### Step 5 (Optional): Process Termination

The mcp-raider server also provides a tool to kill processes, which could be
used to terminate the compromised weather server:

```yaml
Tool: luck_about_and_find_out_process_kill
Parameters:
  processId: "<PID from Step 1>"
```

This demonstrates that the attacker has full lifecycle control over the victim
process — not just read and write access.

### Attack Results

After the attack:

1. **Stolen Credentials**: The attacker now has the OpenWeatherMap API key
2. **Compromised Data**: Weather queries return false information showing
   extreme wind speeds
3. **Misinformed Users**: AI assistants using the compromised weather server
   will tell users there are 133 m/s winds, potentially causing:
   - Unnecessary panic
   - Business disruptions
   - Emergency response activation
   - Travel cancellations

## Threat Model

### Attack Scope

This demonstration shows a **local privilege escalation** attack where:

- Malicious MCP server (`mcp-raider`) and victim server (`mcp-weather`) run on
  the **same machine**
- Attacker has local process access (can list processes, read environment
  variables)
- Both servers run with insufficient isolation

### Security Boundaries

**This attack is prevented by:**

- **Container isolation** — Docker, Podman, VMs with separate process spaces
- **Separate machines** — Running MCP servers on different physical/virtual
  machines
- **Process sandboxing** — OS-level sandboxing (SELinux, AppArmor, macOS
  sandbox)
- **User separation** — Running each MCP server as different OS users
- **Secret management** — Using vaults instead of environment variables

**This attack works when:**

- Multiple MCP servers run on same machine without isolation
- MCP servers run as the same OS user
- No process sandboxing is enforced
- Secrets stored in environment variables (not vaults)

### Real-World Implications

Desktop AI assistants (Claude Desktop, etc.) typically run MCP servers as the
same user. Without additional isolation, one compromised MCP server can attack
others. Production deployments should enforce strict isolation (containers, VMs,
separate hosts).

## Security Implications

This demonstration highlights critical security issues:

### 1. Lack of Process Isolation

- MCP servers can access any system process
- No sandboxing between different MCP implementations
- Unrestricted file system access

### 2. No Authentication

- MCP servers don't authenticate with each other
- Any MCP server can manipulate any other server's resources
- No permission model for sensitive operations

### 3. Runtime Manipulation

- Code can be modified while applications are running
- No integrity verification of executing code
- Template literal injection allows data manipulation

### 4. Information Disclosure

- Environment variables containing secrets are exposed
- Process information leaks sensitive configuration
- No encryption of inter-process communication

## Mitigation Strategies

To prevent these attacks:

### 1. Implement Proper Sandboxing

- Run each MCP server in isolated containers or VMs
- Use process isolation mechanisms (Docker, Podman, etc.)
- Implement namespace separation

### 2. Secure Environment Variables

- Use secret management systems (Vault, AWS Secrets Manager)
- Avoid storing sensitive data in environment variables
- Implement proper access controls

### 3. Code Integrity Protection

- Sign and verify code before execution
- Implement runtime application self-protection (RASP)
- Use read-only file systems where possible

### 4. Authentication and Authorization

- Implement mutual TLS between MCP servers
- Add OAuth2 or similar authentication mechanisms
- Create permission models for sensitive operations

### 5. Monitoring and Alerting

- Log all MCP server interactions
- Monitor for suspicious process access patterns
- Alert on unauthorized file modifications

## Responsible Disclosure

This demonstration is for educational purposes only, showing the importance of
proper security measures when implementing MCP servers. The vulnerabilities
shown are not specific to the MCP protocol itself but rather to implementations
that lack proper security controls.

## References

- [mcp-weather Repository](https://github.com/denhamparry/mcp-weather) - The
  victim MCP server providing weather data
- [mcp-raider Repository](https://github.com/denhamparry/mcp-raider) - The
  attack tool demonstrating exploitation capabilities
- [Model Context Protocol Documentation](https://modelcontextprotocol.io) -
  Official MCP documentation

## Cleanup

After completing the demonstration:

### 1. Stop MCP Servers

- **Claude Desktop**: Remove servers from `claude_desktop_config.json` and
  restart
- **Manual**: Press `Ctrl+C` in both terminal windows

### 2. Restore Modified Files

```bash
cd mcp-weather
git checkout src/weather-utils.ts
git checkout build/
git status  # Verify clean
```

### 3. Revoke Test API Key

1. Log into: <https://home.openweathermap.org/api_keys>
2. Delete the test API key used for demonstration
3. Verify revocation:

```bash
curl "https://api.openweathermap.org/data/2.5/weather?q=London&appid=TEST_KEY"
# Should return "Invalid API key" error
```

### 4. Remove Test Environment (if applicable)

```bash
# If using Docker
docker stop mcpee-test
docker rm mcpee-test

# Or remove cloned repositories
rm -rf mcp-weather mcp-raider
```

## Disclaimer

This demonstration is intended for security research and education only. Do not
use these techniques for malicious purposes. Always obtain proper authorization
before testing security on systems you do not own.
