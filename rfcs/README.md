# RFCs at Fission

This introduction should provide you everything you need to get you started in
the wonderful world of RFCs at Fission. Read on to discover how our organization
handles proposals for additions and changes to the p# RFCs at Fission

This introduction should provide you everything you need to get you started in
the wonderful world of RFCs at Fission. Read on to discover how our organization
handles proposals for additions and changes to the projects and associated
technologies.

### Process

1. **Create a `Draft` RFC based on the provided [template][template]**.

   Create a branch with the title of the RFC (e.g. `Why-Chocolate-Cake-Is-Best`).
   Add a new folder under the `rfcs` directory with the same name as your branch.
   Include a file named `RFC.md` (based on [`template.md`][template]) and
   include any other images/assets that you will use in the write-up within this
   folder. Then, open a [draft PR][draft-prs] against the `main` branch if you
   want anyone to help you prepare it for discussion.
2. **Move to `Discussion`**. Once you have composed a thoughtful proposal that
   addresses the points described in the [template][template], mark the
   [stage][pr-stage] of the PR as `ready for review`. Make sure to select the
   necessary reviewers for your RFC.
3. **Move to `Published`**. If/once the RFC is accepted, then it will be merged
   into `main`. Any merged RFC means that it is `Published`.

_Directory Structure_ example:

```
└── spec
     ├── rfcs
     │    ├── Why-Chocolate-Cake-Is-Best
     │    ├── RFC.md
     │    └── diagrams
     │         └── cake.png
     ├── README.md
     └── template.md
```

### Introduction to RFCs

_RFCs_ (Requests for Comment or sometimes Requests for Discussion) are a method
for communicating and tracking proposals around technical initiatives and
changes within a project.

RFCs provide an opportunity to clarify the goals of a technical proposal before
jumping to an implementation or a versioned specification. Crucially, they
involve explicitly considering alternatives and cross-cutting concerns, and
placing the design in the larger context.

RFCs encourage explicit design discussions, help catch design problems when
change is still (relatively) easy, and ensure that designs adhere to standards
and practices. They form the foundation for strong documentation and
organizational memory.

### When should you create an RFC?

In general, if you are considering an architectural or design decision that
impacts anyone within or outside your team, you should create an RFC.

### When should you not create an RFC?

Generally speaking, you do not need to create an RFC for bugfixes or changes
that are invisible outside of your team. If you’re not sure if your proposed
changes require an RFC, we encourage you to ask for clarification.

#### RFCs vs Technical Specification(s)

A technical specification is any description of a protocol, service,
procedure, convention, or format for implementation and consumption. An RFC, on
the other hand, typically includes more contextual information about the design
and architectural decision(s) being made or can even document a process proposal.
In many cases, the technical spec can live inside of the RFC `Design` section
itself or be linked as a reference.

### What do you need to include in an RFC?

We provide a [template][template] to help you create new RFCs. Be sure you
clearly describe the problem that you are trying to solve in detail, including
requirements and SLAs if warranted. It is impossible to meaningfully evaluate a
proposed solution without this.

There a number of sections which are required to move an RFC from Draft to
Discussion status (i.e. to the point where it will be actively reviewed and
discussed):

- *Introduction* - One concise paragraph describing the proposal.
- *Motivation & Scope* - Why are we doing this and what use cases does it support?
- *Goals & Non-goals* - A short list of bullet points of what the goals of the
  system are, and, sometimes more importantly, what the non-goals are. Non-goals
  are things that could reasonably be goals, but that you're ruling out for
  specific reasons.
- *Design* - The trade-offs you made in considering the design of your software
  and an explanation of a solution that best satisfies the goals you laid out
  above.
- *Alternatives* - What are other possible solutions and why did you rule them
  out?
- *Prior art* - How has this problem been solved or approached before?
- *Unresolved questions* - What questions will be *eventually* be resolved through
  the RFC process, as part of the specification, through the implementation
  process, or only in future work?
- *Impact & Concerns* - How will this have an impact beyond Fission (if it does),
  and what cross-cutting concerns do we need to consider?

The [template][template] includes these sections with more detail about how to
address them.

### On RFC Status

An RFC moves through stages as part of its lifecycle. These are:

- `Draft`: pre-discussion draft of the proposal. Our recommendation is you write
  a short draft as quickly as possible and get your project's members to
  help you shape it into a discussion-ready RFC. Your RFC exists as a
  **draft PR** against `main`.
- `Discussion`: active review and conversation among interested parties. Your
  RFC exists as a **ready to review PR** against `main`.
- `Published`: the RFC has been accepted, merged into `main`, and is ready for
  work.
- `Abandoned`: the RFC had no activity for more than 30 days. The associated PR
  will be marked `Abandoned`.
- `Suspended`: during the RFC process, the engaged parties determined that
  though the proposal might be a good idea, it's not the right time for it. The
  PR can be marked `Suspended` and returned to later.

During the `Draft` and `Discussion` stages of an RFC, it may be helpful to write
prototypes or gists to better convey a line of thinking.

Once an RFC merged into `main`, it's considered `Published`.

### On Timing and Interaction

Once your RFC is ready for discussion, engage with and notify any interested
parties over discord and tag them as the [**required reviewers on a PR**][pr-review].
**Sell what you're building**. If an RFC sits in `Draft` or `Discussion` for
more than 30 days without feedback or clarity, it will be updated with an
`Abandoned` label and the PR will be closed until the author(s) picks it back up
again.

#### For Reference

For more information on how we got to our RFC process, and what we cribbed from,
please check out these references:

- [Design Docs at Google][design-docs-google]
- [RFD 1 Requests for Discussion][oxide-rfd]
- [Rust RFC Process][rust-rfcs]

[design-docs-google]: https://www.industrialempathy.com/posts/design-docs-at-google/
[draft-prs]: https://docs.github.com/en/pull-requests/collaborating-with-pull-requests/proposing-changes-to-your-work-with-pull-requests/about-pull-requests#draft-pull-requests
[oxide-rfd]: https://oxide.computer/blog/rfd-1-requests-for-discussion/
[pr-review]: https://docs.github.com/en/pull-requests/collaborating-with-pull-requests/proposing-changes-to-your-work-with-pull-requests/requesting-a-pull-request-review
[pr-stage]: https://docs.github.com/en/pull-requests/collaborating-with-pull-requests/proposing-changes-to-your-work-with-pull-requests/changing-the-stage-of-a-pull-request
[rust-rfcs]: https://github.com/rust-lang/rfcs
[template]: ./template.md
