## Recommended Pipeline Architecture

### Stage 1: Intake & Triage
- **File type detection**: Use magic number analysis and tools like Detect It Easy to classify incoming files (PE, ELF, Mach-O, MSI, RPM, ZIP, etc.)
- **Risk classification**: Apply YARA rules for initial threat detection and prioritization
- **Metadata extraction**: Capture file hashes, sizes, timestamps, and provenance data for audit trails

### Stage 2: Recursive Extraction Engine
- **Format-specific extractors**: Route packages to appropriate handlers (MSI via `msiexec`, RPM via `rpm2cpio`, ZIP/TAR via standard libraries)
- **Depth limiting**: Prevent extraction bombs with configurable recursion limits (e.g., 5 levels deep)
- **Binary isolation**: Extract only executables and libraries (.exe, .dll, .so, .dylib) while filtering documentation and media files
- **Packer detection**: Identify packed/obfuscated binaries using die-python for secondary unpacking workflows

### Stage 3: Smart Router & MCP Orchestration
Implement centralized governance to route binaries to appropriate MCP servers based on file characteristics:

- **PE files (.exe, .dll)** → **TriageMCP** for Windows binary triage, then dnSpy MCP for .NET assemblies
- **.NET assemblies** → **DecompilerServer** or **mcp-ilspy-server** for CIL decompilation
- **Native ELF/Mach-O** → **binary-mcp** with Ghidra integration for cross-platform analysis
- **Multi-architecture** → B2R2-style parallel lifting for performance at scale

### Stage 4: Analysis Orchestration
Use centralized MCP governance for policy enforcement and credential management:

- **Single authentication**: Agent authenticates once to control plane, receives scoped credentials for MCP servers
- **Policy enforcement**: Define risk tiers (read/mutate/destructive) and apply rate limits per tool
- **Multi-server workflows**: Claude chains calls across multiple MCP servers (e.g., decompile with ILSpy, analyze with YARA, document with filesystem MCP)

### Stage 5: Analysis & Documentation
- **Function-level extraction**: Decompile to C/C#/pseudocode and extract embeddings for similarity analysis
- **Automated documentation**: Use Claude with filesystem MCP to generate analysis reports in Markdown format
- **Vulnerability correlation**: Cross-reference extracted binaries against CVE databases and SBOM data[10]

## Recommended Claude.md Documentation Structure

```markdown
# Binary Analysis Pipeline

## Overview
Multi-stage pipeline for automated extraction and analysis of software packages at scale.

## MCP Server Configuration

### Required Servers
- **binary-mcp**: Multi-tool server (Ghidra + ILSpyCmd + x64dbg)
- **DecompilerServer**: .NET-specific analysis
- **TriageMCP**: PE file triage and YARA scanning
- **filesystem**: Report generation and artifact storage

### Configuration
\`\`\`json
{
  "mcpServers": {
    "binary-mcp": {
      "command": "docker",
      "args": ["run", "-i", "binary-mcp-server"]
    },
    "decompiler": {
      "command": "dotnet",
      "args": ["run", "--project", "DecompilerServer"]
    }
  }
}
\`\`\`

## Workflow Stages

### 1. Extraction Rules
- MSI: Use administrative install method
- RPM/DEB: Extract with package tools
- Nested archives: Max depth = 5 levels
- Filter: Only .exe, .dll, .so, .elf, .dylib

### 2. Routing Logic
```
if file_type == "PE" and is_dotnet:
    route_to(decompiler_server)
elif file_type == "PE":
    route_to(triageMCP) → dnspy_mcp
elif file_type == "ELF":
    route_to(binary_mcp, engine="ghidra")
```

### 3. Analysis Outputs
- Decompiled source code
- Function embeddings for similarity
- YARA match results
- CVE correlation report
```
