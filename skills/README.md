# Skills

One directory per skill, each containing a `SKILL.md`:

    skills/<category>/<skill-name>/SKILL.md

Current categories:

    review-workflow/   the .review/ task-file loop: eject, work, clean up

A directory holding a `SKILL.md` is a skill; a directory that does not is a
category of them. The `skills` CLI walks this tree, so the categories are for
readers of this repo -- they are flattened away on install, and skill names
have to stay unique across categories.

See the top-level `README.md` for install instructions.
