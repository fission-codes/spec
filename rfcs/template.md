# RFC Title

authors:

- `[Author1](Github Link for Author)`

necessary reviewers:

- [ ] Reviewer 1
- [ ] Reviewer 2
- [ ] Reviewer 3

### Introduction

One paragraph explanation of the feature(s), change(s), or addition(s) being
proposed. A diagram can often be helpful here.

---

### Motivation & Scope

Clearly describe the problem that you are trying to solve in detail, including
requirements. It is impossible to meaningfully evaluate a proposed solution
without this.

Some of the questions to consider here:

- Why are we doing this?
- What use cases does it support?
- What is the expected outcome?
- What are known or expected SLAs?
- What's needed for minimum viability?

---

### Goals & Non-goals

A short list of bullet points of what the goals of the system are, and,
sometimes more importantly, what the non-goals are.

**Note**: Non-goals aren’t obviously bad things like "A system that crashes all
the time", but rather things that could reasonably be goals, but are explicitly
chosen not to be goals. A good example would be "ACID compliance"; when
designing a database, you’d certainly want to know whether that is a goal or
non-goal. And if it is a non-goal, you might still select a solution that
provides it, as long as it doesn’t introduce trade-offs that prevent achieving
the goals.

---

### Design

This is the technical portion of the RFC where you'll write down the trade-offs
you made in considering the design of your software and suggest a solution that
best satisfies the goals you laid out above.

Explain the design in sufficient detail that:

- Its interaction with other applications, frameworks, systems, etc. is made
  clear. Diagrams are very helpful here.
- It is reasonably clear how one would go about implementing your solution from
  this design.
- Corner cases are dissected by example.

**Note**: Don't use this as a direct implementation manual. If something can be
implemented straightforwardly, then it probably doesn't need to coincide with an
RFC.

---

### Alternatives

- What other designs have been considered and what is the rationale for not
  choosing them?
- What if we didn't commit to this proposal or design? What else could we do? Is
  there another solution with specific trade-offs?
- It can be useful to explicitly consider "do nothing" as an alternative.

---

### Prior Art

Discuss prior art, both the good and the bad, in relation to this proposal. A
few examples of what this can include are:

- Does this feature, change, addition exist in other systems, platforms,
  languages, libraries? How has it been used?
- What lessons can we learn from what other companies and/or communities have
  done here?
- Papers or posts: Are there any published papers or great posts that discuss
  this? If you have some relevant papers to refer to, this can serve as a more
  detailed background.

This section is intended to encourage you as an author to think about the
lessons learned from what others have done as well as provide readers of your
RFC with a fuller picture. If there is no prior art, that is fine - your ideas
are interesting nonetheless!

---

### Unresolved questions

- What parts of the design do you expect to resolve through the RFC process
  before this gets merged?
- What parts of the design do you expect to resolve through the implementation
  of this feature, change, or addition before going into production?
- What related issues do you consider out of scope for this RFC that could be
  addressed in the future, independently of the solution that comes out of this
  RFC?

---

### Impact & Concerns

This section is intended to encourage you to think about and make sure that
cross-cutting concerns, such as security, privacy, and observability, are always
taken into consideration.

Also, it gives you a chance to mention how your design impacts the organization,
implementers, and/or consumers going forward.
