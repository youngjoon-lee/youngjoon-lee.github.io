---
layout: post
title: "Published an open source Sphinx packet implementation"
date:   2024-01-13 15:50:00 +09:00
---

I just published an open source implementation of Sphinx packet, [pysphinx](https://pypi.org/project/pysphinx/), that can be used for mix network projects like [Nym](https://github.com/nymtech/nym).

This implementation complies with the [standard Sphinx packet specification](https://cypherpunks.ca/~iang/pubs/Sphinx_Oakland09.pdf).

The initial version v0.0.1 provides most of basic functionalities:
- Constructing a Sphinx packet with encapsulations (encryption)
- Deconstructing a Sphinx packet

The v0.0.1 doesn't support the following features yet:
- Specifying delays and SURB identifiers when constructing Sphinx packets: https://github.com/youngjoon-lee/pysphinx/issues/2
- Payload encryption: https://github.com/youngjoon-lee/pysphinx/issues/4
