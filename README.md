# Adrien Gentil's Claude Skills

A collection of productivity skills for Claude Code to enhance GitHub workflows and project management.

## Skills

### pr-dashboard

List and classify open PRs from a GitHub organization updated in a specified time range (default 1 day), prioritizing those waiting for your review, with approval recommendations.

**Usage:**
```
/pr-dashboard <organization-name> [days]
```

**Examples:**
```
/pr-dashboard osac-project           # Last 24 hours (default)
/pr-dashboard osac-project 3         # Last 3 days
/pr-dashboard osac-project 7         # Last week
```

## Installation

To use these skills in Claude Code, add this repository to your skills sources:

1. Open Claude Code settings
2. Navigate to Skills
3. Add this repository URL: `https://github.com/adriengentil/skills`

## Contributing

Feel free to open issues or submit PRs to add new skills or improve existing ones.

## License

MIT
