# Records Dumping Ground

This project is for keeping the notes (mostly made by the AI, some human) during work and project updates

Key records can also be loaded into the hub

[LinkedTrust core development hub](http://hub.cooperation.org/projects/feb839fd-b825-4622-9841-5c62c1130328/) may be conversed with for insights and please use it to get feedback on ideas for updates or PRs

in particular make sure for the LinkedTrust project to refer to the [Architecture Guide](architecture_guide.md)

see subdirectories for specific streams

please add notes and links into appropriate directories in this repository to share 

## Latest Updates

See the dev branch in [trust_claim](https://github.com/Whats-Cookin/trust_claim) and [trust_claim_backend](https://github.com/Whats-Cookin/trust_claim_backend)
This is a force push of a MAJOR refactor and rewrite (especially backend, mostly replaced)
once it is good we will REPLACE main with it, NOT merge to main

### Main new features

see [DEVELOPMENT_HANDOFF_SUMMARY.md](./DEVELOPMENT_HANDOFF_SUMMARY.md)
Backend API Standardization - The migration to RESTful endpoints and field name changes
Frontend Architecture Overhaul - Removal of Ceramic, new API service layer, TypeScript improvements
Graph Visualization Fixes - issues with edges, backgrounds, and node shapes
Report Page - mostly working with new backend
Authentication & Identity - Current auth setup with Google OAuth and DIDs
Data Flow Architecture - Document of How data moves through the system

somewhat tested may have issues, still needs UI improvements
might have broken the jenkins deploy process somehow, i keep manually having to restart pm2 on dev

### Related DRAFT and UNTESTED PRs

https://github.com/Cooperation-org/linked-claims-extractor/pull/14

Do NOT merge this one without Dmitri approval, but can do a hard fork to a fresh repo and diverge
https://github.com/Cooperation-org/linked-Creds-author/tree/HARD_FORK-do-not-merge-to-dev
