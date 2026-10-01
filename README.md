# CrossCentral

National-level application of the Czech Red Cross (ČČK). It has two jobs:

- **Federation hub.** A registry of every
  [MemberBase](https://github.com/spidermila/MemberBase) deployment in the
  country (one per Oblastní spolek), and the channel for predefined requests
  between them and from national bodies to them. A person in the source
  Oblastní spolek approves every request before any member data leaves it.
- **National agenda.** The internal work of the national level, starting with
  a register of the people who use CrossCentral and their roles. A knowledge
  base, a brand repository and a logo generator may follow.

The UI is Czech; code and docs are English.

## Status

Design phase; there is no code yet. Requirements, architecture decisions and
open questions are in [`architecture.md`](architecture.md).

## Related projects

- [MemberBase](https://github.com/spidermila/MemberBase) – member directory
  of one Oblastní spolek („Evidence členů“).
- [MedCover](https://github.com/spidermila/MedCover) – event medical cover
  staffing, built by the same team.
