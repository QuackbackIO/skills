# Quackback skills

Agent skills for installing Quackback in a host codebase.

## Install as a marketplace

```
/plugin marketplace add QuackbackIO/skills
/plugin install install-widget@quackback-skills
```

## Install the widget skill by hand

Copy the skill into the host repo so the agent can see it:

```bash
mkdir -p .agents/skills
git clone --depth 1 https://github.com/QuackbackIO/skills.git /tmp/quackback-skills
cp -R /tmp/quackback-skills/skills/quackback/install-widget .agents/skills/install-widget
```

Or tell the agent to fetch and follow:

- https://raw.githubusercontent.com/QuackbackIO/skills/main/skills/quackback/install-widget/SKILL.md
- https://raw.githubusercontent.com/QuackbackIO/skills/main/skills/quackback/install-widget/references/identify-users.md

Workspace URL and widget secret come from **Admin → Settings → Widget → Install**. Do not commit the secret.

## License

MIT — see [LICENSE](LICENSE).
