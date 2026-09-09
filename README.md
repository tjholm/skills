# skills

Agent skills for Claude Code and other agents that read the same format.

    skills/   one directory per skill, each holding a SKILL.md,
              grouped into category directories

## Install

Use the [`skills` CLI](https://github.com/vercel-labs/skills). To see what is
in here without installing anything:

    npx skills add tjholm/skills --list

Install everything, globally, for whichever agents it finds:

    npx skills add tjholm/skills --global

Or pick individual skills by name:

    npx skills add tjholm/skills --skill next-review-task --skill eject-problem-set

Drop `--global` to install into the current project instead of your home
directory, and pass `--agent claude-code` to target one agent rather than being
prompted. The CLI symlinks by default, so `npx skills update` picks up changes
here; use `--copy` if you would rather have your own copy. `npx skills remove`
takes them back out.

The three `review-workflow` skills are one loop and are meant to be installed
together: `eject-problem-set` splits a findings list into task files under
`.review/`, `next-review-task` works one of them per session, and
`clean-review-tasks` prunes the ones that are done.

## License

MIT -- see `LICENSE.md`.
