# Design Docs

Welcome to the design docs repository. This repo is used to coordinate
around engineering design and risk analysis.

## Contributing

Please use the templates found in the `assets` directory when starting
a document. Place the completed document in the appropriate top level
directory after it is completed.

- `protocol` contains consensus level features
- `ecopod` contains application level features
- `governance` contains governance level features
- `security` contains failure mode analysis documents

## Design Review Sessions

The agendas for design doc review sessions are being tracked on 
Github [here](https://github.com/ethereum-optimism/design-docs/issues/15).
Please comment on the Github issue that corresponds to the design
review session that you would like to have your design doc included in.
Please tag particular individuals that you would like to be reviewers
of the document. Its good to tag people with high context of the components
being modified or owners of interfacing components. Be sure to reach out to
the reviewers and get explicit acknowledgement from them so that they can
review the design doc ahead of time.

The agenda for a design doc review session is locked in 24 hours before the
meeting. Each Design Doc must be reviewed asynchronously prior to the meeting.
Comments on the PR are encouraged. Refer to the following SLA for required lead
time between opening the PR and it getting onto the Design Review agenda:

- Short document (e.g., 1-pager): share a minimum of 2 working days prior
- Medium-length document (2-5 pages): share a minimum of 3-5 working days prior
- Long document (6+ pages): share at least 1 week prior

The goal of the discussion is to move towards closure, where closure is either
ratifying or rejecting the Design Doc under review. This means that each discussion
will end one of the following three ways:

- The doc is ratified, in which case the PR is merged and the decision is considered made.
- The doc is rejected, in which case the PR is closed with a comment explaining why.
- The team requests more info, in which case the PR is left open for discussion during the following week’s design review.
