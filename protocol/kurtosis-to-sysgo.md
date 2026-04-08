# Migrating from Kurtosis to `sysgo`: Design Doc

|                    |                                                    |
| ------------------ | -------------------------------------------------- |
| Author             | Josh Klopfenstein, Matthew Slipper                 |
| Created at         | 2025-09-09                                         |
| Initial Reviewers  | _TBD_                                              |
| Need Approval From | _TBD_                                              |
| Status             | Draft <!--/ In Review / Implementing Actions / Final_ --> |

## Summary

We currently use both `sysgo` and Kurtosis to deploy local OP Stack devnets.

**To consolidate engineering efforts, this design doc proposes deprecating Kurtosis in favor of exclusively using `sysgo`, while providing a migration path for existing Kurtosis consumers.**

It is worth noting that the `netchef` tool continues to be useful for deploying persistent, remote devnets, but is out of scope of this document. The `op-up` tool is also out-of-scope because it is built on `sysgo` and primarily focuses on top-of-funnel developers rather than power users. Both are mentioned in the “Alternatives Considered” section.

## Problem Statement + Context

<!-- Describe the specific problem that the document is seeking to address as well
as information needed to understand the problem and design space.
If more information is needed on the costs of the problem,
this is a good place to that information. -->

It is costly to maintain both Kurtosis and `sysgo` because both fulfill the same purpose (deploying local devnets) while requiring separate skillsets to modify and use. They are orthogonal implementations of the same concept:

|                          | **Kurtosis**                                              | **`sysgo`** |
| ------------------------ | --------------------------------------------------------- | --- |
| **Backend**                  | Docker + Kurtosis enclaves                                | Spawn Go services in-process, spawn non-Go services as subprocesses |
| **Implementation language**  | Starlark                                                  | Go |
| **Configuration format**     | Bespoke yaml DSL                                          | Go |
| **Third-party dependencies** | Kurtosis itself, `ethereum-package` Starlark code, Docker | N/A |

Redundant tooling also creates an unnecessary cognitive burden for consumers, who must learn their way around additional idiosyncrasies.

## Proposed Solution

<!-- A high level overview of the proposed solution.
When there are multiple alternatives there should be an explanation
of why one solution was picked over other solutions.
As a rule of thumb, including code snippets (except for defining an external API)
is likely too low level. -->

We propose deprecating Kurtosis in favor of `sysgo`.

### Rationale

The requirements for a local devnet solution are outlined below.

|     | **Kurtosis** | **`sysgo`** | **Priority** |
| --- | ------------ | ----------- | ------------ |
| **Compile-time extensibility** (can I define new services that will be deployed with the rest of the system?) | Possible, but requires building a separate Kurtosis (Starlark) package or extending the existing one. | Relatively easy: a few dozen lines of Go is usually all that’s needed. | P0 |
| **Runtime extensibility** (can I deploy new services or connect to existing services after the system is deployed?) | Quite extensible due to deterministic port assignment. | Not extensible once the system is running: ports are unknown and generally hidden behind opaque interfaces. The opaque interfaces are usually all you need when inside the original Go process, however. | P2 |
| **Startup time** | 1-5 minutes | <1 minute | P0 |
| **Reliability and debug-ability**  | Sometimes fails for obscure reasons; debugging requires Kurtosis expertise. | Almost always succeeds; any Go programmer can debug with standard tools and practices. | P0 |
| **Installation and setup** (important for development outside the monorepo) | Docker images. | Bespoke: any mix of `go get`, Cargo, mise could work. If using multiple versions of the same service, mise may be required. | P2 |
| **Metrics visualization** | Supported via Prometheus and Grafana; distributes some OP dashboards out of the box | Not supported (yet) | P0 |

In terms of developer experience, the most important features are startup time and compile-time extensibility. In these categories, `sysgo` is a clear winner. Kurtosis’s heavy use of Docker and Starlark abstractions render it fundamentally limited in these areas.

The Kurtosis features that `sysgo` currently lacks, such as metrics visualization, can be added or worked-around without too much trouble if needed.

### Migrating from Kurtosis

The single P0 requirement gap between Kurtosis and `sysgo` is metrics visualization using Prometheus and Grafana. To fill this gap, the `sysgo` orchestrator can track metrics endpoints during setup before passing those endpoints to Prometheus for scraping. To enable Prometheus and Grafana, the caller can pass in a `WithMetrics()` option.

This already goes a long way toward supporting existing Kurtosis consumers:

| **Developer** | **Support Required** |
| --- | --- |
| Teams developing for the OP Stack outside of the monorepo (Rust team, [flashblocks/rollup-boost](https://github.com/flashbots/rollup-boost/blob/0ce4158bc32f8405458c506ca38ad5d54aa81949/docs/local-devnet.md?plain=1#L11), etc.) | Provide documentation and examples for replacing Kurtosis with `sysgo`. It will require installing Go and writing small Go programs or tests. It also needs to be straightforward to obtain contract artifacts (for `op-deployer`) and prestates (for `op-challenger`). |
| Developers using Kurtosis for visualizing metrics on local devnets | Implement metrics visualization in `sysgo`. |

Kurtosis should continue to work with its existing features for the foreseeable future. Migrating is not time-sensitive if there isn’t a dependency on new OP Stack components or features.

## Failure Mode Analysis

<!-- Link to the failure mode analysis document, created from the fma-template.md file. -->

N/A. See “Risks & Uncertainties”.

## Impact on Developer Experience
<!-- Does this proposed design change the way application developers interact with the protocol?
Will any Superchain developer tools (like Supersim, templates, etc.) break as a result of this change? -->

Hopefully positive.

## Alternatives Considered

<!-- List out a short summary of each possible solution that was considered.
Comparing the effort of each solution -->

Another option is to go all-in on `netchef`, using it for local devnets in addition to remote devnets (i.e., making it possible to deploy the generated k8s resources into minikube rather than the cloud). It would be more extensible than `sysgo` and would allow us to combine engineering efforts even further, but its startup time would be greater.

Alternatively, we could take inspiration from `netchef` and implement a Docker compose backed in `op-up`: generating a `docker-compose.yaml` file based on some high-level configuration provided by the user. This effectively replaces Kurtosis with another backend, albeit a far simpler one. It would probably be more extensible and have a reasonable startup time, but would continue to divide our development efforts.

## Risks & Uncertainties

<!-- An overview of what could go wrong.
Also any open questions that need more work to resolve. -->

Whenever we aim to consolidate engineering efforts and bless a single tool or process, there is a risk that we introduce “yet another standard” <insert obligatory xkcd>. Because `sysgo` already exists, this risk is addressed: we are strictly eliminating requirements from the critical path.

Second, there is risk of complexity creep in `sysgo` due to implementing additional features. This is deemed to be an acceptable trade-off that can be effectively mitigated by maintaining high code-quality standards.

Third, closing the gaps between `sysgo` and Kurtosis may take longer than estimated.

Finally, there is a risk that we continue to half-heartedly support Kurtosis, rather than making a clean cut and investing fully in `sysgo`. If this design doc is accepted, it is assumed that there will be sufficient organizational willpower to finish its implementation. Otherwise, the proposal to deprecate Kurtosis may be misguided.
