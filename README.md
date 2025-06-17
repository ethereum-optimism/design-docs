# Design Docs

Welcome to the design docs repository. This repo is used to coordinate
around engineering design and risk analysis.

The design review process is important for sharing context and aligning
on the future of the OP Stack in a public and transparent way. A good design
doc will explain why a particular solution solves a problem well by explaining
necessary context. It can include pseudocode and diagrams but does not need
to be as specific as a spec, which should include all information required
to implement the feature. It is important to include product perspective
in the design review process, given we want people to actually use what we are building.

## Contributing

Please use the templates found in the `assets` directory when starting
a document. Place the completed document in the appropriate top level
directory after it is completed.

- `protocol` contains consensus level features
- `ecopod` contains application level features
- `governance` contains governance level features
- `security` contains failure mode analysis documents

[`doctoc`](https://www.npmjs.com/package/doctoc) is used for maintaining tables of contents.
For an example of usage, see the [FMA Template](https://github.com/ethereum-optimism/design-docs/blob/main/assets/fma-template.md?plain=1).

Usage:
1. `npm install -g doctoc`
2. `doctoc <filename>`

## Design Review Sessions

The agendas for design doc review sessions are being tracked on 
Github [here](https://github.com/ethereum-optimism/design-docs/issues/15).

To schedule a design review sesion:
- Ensure that you have a pull request that has been reviewed async
- [Open an issue](https://github.com/ethereum-optimism/design-docs/issues/new/choose) and click "Schedule a Design Review" then fill in the information. Please submit your request at least 72 hours before your meeting time to ensure it is posted in time.
- A design review meeting will be added to the [Public OP Stack Calendar](https://calendar.google.com/calendar/embed?src=c_e7b35eadabec39777b28192d371c45b6ef4177e01740517a234e7c768881fbfe%40group.calendar.google.com&ctz=America%2FLos_Angeles) by an admin. Please reach out on discord if this hasn't happened. 

When choosing which stakeholders to invite to a design review meeting,
it is good to tag people with high context of the components
being modified or owners of interfacing components. Be sure to reach out to
the reviewers and get explicit acknowledgement from them so that they can
review the design doc ahead of time.

The goal of the discussion is to move towards closure, where closure is either
ratifying or rejecting the Design Doc under review. This means that each discussion
will end one of the following three ways:

- The doc is ratified, in which case the PR is merged and the decision is considered made.
- The doc is rejected, in which case the PR is closed with a comment explaining why.
- The team requests more info, in which case the PR is left open for discussion during the following week’s design review.

The agenda for a design doc review session is locked in 24 hours before the
meeting. Each Design Doc must be reviewed asynchronously prior to the meeting.
Comments on the PR are encouraged. Refer to the following SLA for required lead
time between opening the PR and it getting onto the Design Review agenda:

- Short document (e.g., 1-pager): share a minimum of 2 working days prior
- Medium-length document (2-5 pages): share a minimum of 3-5 working days prior
- Long document (6+ pages): share at least 1 week prior
