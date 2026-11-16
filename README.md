# project_manager — Hermes profile distribution

Generated 2026-11-16T16:35:49+03:30 from /home/siavash/.hermes/profiles/project_manager by to-profile-dist.sh.

Install on any machine:
    hermes profile install <this-repo-git-url>
    cp ~/.hermes/profiles/project_manager/.env.EXAMPLE ~/.hermes/profiles/project_manager/.env   # fill keys, chmod 600
    hermes -p project_manager chat -Q -q "reply with exactly: PROFILE-OK"

Iterate: edit the files here, bump version in distribution.yaml, push; then on each machine:
    hermes profile update project_manager
User data (memories, sessions, .env, auth) is never overwritten by updates;
config.yaml is preserved unless --force-config.
