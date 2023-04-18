# :cherries: Capability Hardware Enhanced RISC Instructions (CHERI)

!!! warning "This Chapter is Work In Progress"

This is the landing page for CHERI, a new computer architecture designed to enable fine-grained memory protection and highly scalable software compartmentalization.

This page provides you only with the definition of what CHERI is and what it does. To properly get acquianted with the topic, start by reading the [Motivation](./motivation.md) article. Thereafter, the article on the [Technical Background](./technical_background.md) provides you with detailed information on how CHERI works. This article is closely connected to the article on [Capabilities](./capabilities.md). In the end, the article on [Consequences](./consequences.md) goes into detail about where and how CHERI can be applied.

!!! info "Definition: Capability Hardware Enhanced RISC Instructions (CHERI)"

    ~ hybrid capability architecture extension able to blend architectural capabilities with conventional MMU-based architectures. It extends conventional ISAs, relying on architectural capabilities to enforce

    1. fine-grained memory protection;
    2. strong, non-probabilistic, efficient mechanisms which support the principles of least privilege (POLA);
    3. the intentional use in the execution of software at multiple levels of abstraction.
