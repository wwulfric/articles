# GitHub Copilot Coding Agent Configuration

This directory contains configuration files for GitHub Copilot coding agent.

## Internet Access Configuration

The `workflows/copilot-setup-steps.yml` file configures the ephemeral development environment used by GitHub Copilot coding agent when it works on issues.

### How It Works

When Copilot coding agent works on an issue, it:
1. Spins up an isolated ephemeral environment (using GitHub Actions)
2. Runs the setup steps defined in `copilot-setup-steps.yml`
3. Has access to the configured environment and internet resources

### Customizing Network Access

By default, Copilot coding agent has limited internet access through a built-in firewall. To customize this:

1. **Via Repository Settings:**
   - Go to repository Settings → Coding agent
   - Add custom hosts/domains to the allowlist
   - Configure firewall settings as needed

2. **Via Workflow Configuration:**
   - Modify `.github/workflows/copilot-setup-steps.yml`
   - Add environment variables
   - Install additional tools or dependencies

### Security Considerations

- The firewall helps prevent data exfiltration and unintentional information leaks
- Review and approve any changes to network access configuration
- Use the recommended allowlist for common package managers and registries

### References

- [Customizing the development environment for GitHub Copilot coding agent](https://docs.github.com/en/copilot/how-tos/use-copilot-agents/coding-agent/customize-the-agent-environment)
- [Customizing or disabling the firewall for GitHub Copilot coding agent](https://docs.github.com/en/copilot/how-tos/use-copilot-agents/coding-agent/customize-the-agent-firewall)
