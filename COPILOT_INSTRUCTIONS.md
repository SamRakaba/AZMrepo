# Copilot Instructions for AZMrepo Project

This document provides specific instructions and guidelines for using GitHub Copilot effectively within the AZMrepo project for agentic Azure Management (AZM) consolidation and generation.

## Project Overview

**Project Name**: AZMrepo
**Purpose**: Agentic Azure Management consolidation and generation spreadsheet
**Description**: This repository contains tools and documentation for building Copilot Studio agents focused on Azure Management tasks.

## Copilot Configuration

### General Guidelines

When working with this repository, GitHub Copilot should:

1. **Focus on Azure Management Context**: All suggestions should consider Azure resource management, automation, and monitoring scenarios
2. **Spreadsheet Integration**: Consider data consolidation and generation patterns suitable for spreadsheet formats
3. **Agent-First Approach**: Prioritize conversational AI patterns and agent-based solutions
4. **Documentation-Driven**: Emphasize clear, comprehensive documentation for all features

### Code Style Preferences

- **Language**: Prefer Markdown for documentation, Python for scripting, PowerShell for Azure automation
- **Formatting**: Use clear headings, bullet points, and structured formats
- **Comments**: Include inline comments for complex logic
- **Naming**: Use descriptive, self-documenting names (e.g., `consolidate_azure_resources` not `car`)

### Documentation Standards

When generating documentation:

1. **Structure**: Always include:
   - Clear title and purpose
   - Table of contents for long documents
   - Prerequisites section
   - Step-by-step instructions
   - Examples and code snippets
   - Troubleshooting section
   - Resources and references

2. **Format**: Use:
   - H1 (#) for main titles
   - H2 (##) for major sections
   - H3 (###) for subsections
   - Code blocks with language specifications
   - Bullet points for lists
   - Numbered lists for sequential steps

3. **Examples**: Provide:
   - Real-world scenarios
   - Complete, runnable code samples
   - Expected outputs
   - Common variations

### Azure-Specific Patterns

When suggesting Azure-related code or documentation:

1. **Resource Management**:
   ```python
   # Example: Copilot should suggest patterns like this
   from azure.mgmt.resource import ResourceManagementClient
   from azure.identity import DefaultAzureCredential
   
   # Use DefaultAzureCredential for authentication
   credential = DefaultAzureCredential()
   resource_client = ResourceManagementClient(credential, subscription_id)
   ```

2. **Error Handling**:
   ```python
   # Always include proper error handling for Azure operations
   try:
       resources = resource_client.resources.list()
   except Exception as e:
       logging.error(f"Failed to list resources: {str(e)}")
       raise
   ```

3. **Cost Awareness**: Suggest solutions that consider Azure cost implications

### Spreadsheet Generation Patterns

When suggesting code for spreadsheet operations:

1. **Prefer Pandas**: Use pandas DataFrames for data manipulation
   ```python
   import pandas as pd
   
   # Create structured data
   df = pd.DataFrame(resources_data)
   df.to_excel('azure_resources.xlsx', index=False)
   ```

2. **Excel Formatting**: Include formatting suggestions for readability
   ```python
   # Add formatting to Excel output
   with pd.ExcelWriter('output.xlsx', engine='openpyxl') as writer:
       df.to_excel(writer, sheet_name='Resources', index=False)
       worksheet = writer.sheets['Resources']
       worksheet.column_dimensions['A'].width = 20
   ```

3. **Data Validation**: Include data validation and cleaning steps

### Copilot Studio Agent Patterns

When suggesting agent-related content:

1. **Topic Structure**: Follow the pattern:
   - Clear trigger phrases (5-10 variations)
   - Greeting and context setting
   - Information gathering with questions
   - Action execution or response
   - Confirmation and next steps

2. **Conversation Flow**:
   ```
   User: "Show me my Azure VMs"
   Agent: "I'll help you view your Azure VMs. For which subscription?"
   User: "Production subscription"
   Agent: [Execute action to list VMs]
   Agent: "Here are your VMs in Production: [List]. Would you like details on any?"
   ```

3. **Error Responses**: Always include user-friendly error handling:
   ```
   "I encountered an issue retrieving your Azure resources. 
   Please verify:
   - You have the necessary permissions
   - Your subscription is active
   - The resource name is correct
   Would you like to try again or speak with support?"
   ```

### Testing Expectations

When suggesting test code:

1. **Unit Tests**: Include tests for individual functions
2. **Integration Tests**: Test Azure API interactions with mocks
3. **Documentation Tests**: Verify documentation examples work
4. **Agent Flow Tests**: Test conversation scenarios end-to-end

### Security Considerations

Copilot should always suggest:

1. **Credential Management**: Use Azure Key Vault or environment variables, never hardcode
2. **Least Privilege**: Suggest minimal required permissions
3. **Data Protection**: Handle sensitive Azure resource data appropriately
4. **Audit Logging**: Include logging for compliance
5. **Input Validation**: Validate all user inputs in agent conversations

### Common Patterns for This Project

#### 1. Azure Resource Consolidation

```python
def consolidate_azure_resources(subscription_id: str, resource_types: list) -> pd.DataFrame:
    """
    Consolidate Azure resources into a structured DataFrame.
    
    Args:
        subscription_id: Azure subscription ID
        resource_types: List of resource types to include
        
    Returns:
        DataFrame with consolidated resource information
    """
    # Implementation here
    pass
```

#### 2. Spreadsheet Generation

```python
def generate_resource_spreadsheet(resources: pd.DataFrame, output_path: str) -> None:
    """
    Generate formatted Excel spreadsheet from Azure resources.
    
    Args:
        resources: DataFrame containing resource data
        output_path: Path for output Excel file
    """
    # Implementation here
    pass
```

#### 3. Agent Response Templates

```markdown
### Template: Resource Status Response

**User Query**: "What's the status of my Azure resources?"

**Agent Response Structure**:
1. Acknowledge request
2. Specify scope (subscription/resource group)
3. Provide summary statistics
4. Offer detailed breakdown
5. Suggest next actions
```

### File Organization

Suggested file structure that Copilot should follow:

```
AZMrepo/
├── README.md                          # Project overview
├── COPILOT_STUDIO_GUIDE.md           # How to build agents
├── COPILOT_INSTRUCTIONS.md            # This file
├── docs/                              # Additional documentation
│   ├── azure_setup.md                 # Azure configuration guide
│   ├── agent_topics.md                # Agent conversation topics
│   └── api_reference.md               # API documentation
├── scripts/                           # Automation scripts
│   ├── consolidate_resources.py       # Resource consolidation
│   └── generate_spreadsheet.py        # Spreadsheet generation
├── agents/                            # Agent definitions
│   ├── azure_vm_agent.json           # VM management agent
│   └── cost_analysis_agent.json       # Cost analysis agent
├── templates/                         # Spreadsheet templates
│   └── azure_resources_template.xlsx
└── tests/                             # Test files
    ├── test_consolidation.py
    └── test_agents.py
```

### Prompt Engineering Tips

When working on agent conversation design in this project:

1. **Be Specific**: "List all VMs in the Production subscription" (not "show VMs")
2. **Provide Context**: Always specify subscription, resource group, or region
3. **Use Natural Language**: "Show me cost breakdown for last month" (natural)
4. **Include Examples**: Provide sample inputs and expected outputs
5. **Handle Ambiguity**: Design for multiple interpretations

### Variable Naming Conventions

- **Subscriptions**: `subscription_id`, `sub_id`
- **Resource Groups**: `resource_group_name`, `rg_name`
- **Resources**: `resource_name`, `resource_id`
- **DataFrames**: `df_resources`, `df_costs`, `df_summary`
- **Agent Variables**: `user_subscription`, `selected_resource`, `operation_type`

### Dependencies to Suggest

For Azure operations:
```
azure-identity
azure-mgmt-resource
azure-mgmt-compute
azure-mgmt-storage
azure-mgmt-monitor
```

For data processing:
```
pandas
openpyxl
xlsxwriter
```

For agent development:
```
requests
python-dotenv
pyyaml
```

### Git Commit Message Format

Copilot should suggest commit messages in this format:
```
<type>: <subject>

<body>

<footer>
```

Types: `feat`, `fix`, `docs`, `style`, `refactor`, `test`, `chore`

Examples:
- `docs: Add Copilot Studio setup instructions`
- `feat: Implement Azure VM consolidation script`
- `fix: Handle missing subscription ID error in agent`

### Code Review Checklist

Before suggesting code is complete, ensure:

- [ ] Follows Azure best practices
- [ ] Includes error handling
- [ ] Has documentation/comments
- [ ] Uses proper authentication
- [ ] Validates inputs
- [ ] Logs important operations
- [ ] Handles edge cases
- [ ] Includes usage examples
- [ ] Follows security guidelines
- [ ] Is testable

### Agent-Specific Guidance

For Copilot Studio agent development in this project:

1. **Knowledge Sources**: Suggest connecting to:
   - Azure documentation
   - Internal Azure resource wikis
   - Cost management reports
   - Compliance documentation

2. **Common Topics to Implement**:
   - List Azure resources
   - Check resource status
   - Analyze costs
   - Generate reports
   - Troubleshoot issues
   - Provide recommendations

3. **Integration Actions**: Suggest:
   - Power Automate flows for Azure operations
   - Azure Functions for complex logic
   - Logic Apps for workflows
   - Azure DevOps for automation

### Performance Considerations

When suggesting solutions:

1. **Caching**: Cache Azure resource data to reduce API calls
2. **Pagination**: Handle large result sets with pagination
3. **Async Operations**: Use async patterns for concurrent operations
4. **Rate Limiting**: Respect Azure API rate limits
5. **Batch Operations**: Batch similar operations when possible

### Localization

If adding multi-language support:

1. Use resource files for strings
2. Support common Azure regions
3. Handle currency formatting for costs
4. Consider time zone differences

### Accessibility

Ensure agent responses are:

1. Clear and concise
2. Structured with proper formatting
3. Accessible to screen readers (when UI is involved)
4. Available in multiple formats (text, tables, charts)

## Example Scenarios

### Scenario 1: VM Status Check

**User Input**: "Show me status of all VMs in production"

**Expected Agent Flow**:
1. Validate user authentication
2. Identify "production" subscription/resource group
3. Query Azure for VM status
4. Format results in readable format
5. Offer drill-down options

**Code Pattern**:
```python
def get_vm_status(subscription_id: str, resource_group: str = None) -> pd.DataFrame:
    """Get status of all VMs in scope."""
    # Implementation
    pass
```

### Scenario 2: Cost Report Generation

**User Input**: "Generate cost report for last month"

**Expected Agent Flow**:
1. Determine date range
2. Fetch cost data from Azure Cost Management
3. Aggregate by service/resource group
4. Generate Excel report
5. Provide download link or summary

**Code Pattern**:
```python
def generate_cost_report(subscription_id: str, start_date: str, end_date: str) -> str:
    """Generate Excel cost report and return file path."""
    # Implementation
    pass
```

## Maintenance Notes

- Review and update these instructions quarterly
- Align with latest Azure and Copilot Studio features
- Incorporate feedback from project team
- Update patterns based on actual usage

## Additional Resources

- [GitHub Copilot Documentation](https://docs.github.com/en/copilot)
- [Azure Python SDK Documentation](https://docs.microsoft.com/en-us/python/api/overview/azure/)
- [Copilot Studio Documentation](https://learn.microsoft.com/en-us/microsoft-copilot-studio/)
- [Azure Best Practices](https://learn.microsoft.com/en-us/azure/architecture/best-practices/)

---

**Version**: 1.0
**Last Updated**: February 2026
**Maintained By**: AZMrepo Project Team
