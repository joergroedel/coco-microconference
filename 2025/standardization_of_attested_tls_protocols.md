# Standardization of Attested TLS Protocols ([Video](https://www.youtube.com/watch?v=MF9AwkMJOlw))

Presenter: Muhammad Usama Sardar

The video session, titled **"Standardization of Attested TLS Protocols,"** was
presented by Muhammad Usama Sardar from TU Dresden at the 2025 Linux Plumbers
Conference in Tokyo. The presentation focused on integrating remote attestation
into TLS to support Confidential Computing (CC) environments, particularly for
sensitive data like genomic and health records.

## **Presentation Summary**

- **System Model and Goals:** Sardar outlined a model where a TLS server acts
  as a Remote Attestation (RATS) attester. This involves a complex infrastructure
  including a CC platform, a quoting agent, and virtual machines. The primary
  security goals are to ensure the **integrity and freshness** of attestation
  evidence and to **strongly bind** this evidence to a specific TLS session to
  prevent relay attacks.
- **The "Big Picture" of Standardization:** He detailed the involvement of
  multiple working groups, such as the IETF's TLS and SEAT groups, in defining
  how attestation should be integrated. The discussion covered three main
  approaches:
  - **Pre-handshake:** Attestation occurs before the TLS connection is
    established (e.g., Intel’s RA-TLS).
  - **Intra-handshake:** Attestation evidence is embedded within the TLS
    handshake messages.
  - **Post-handshake:** Attestation happens after a secure channel is
    established, allowing for continuous or on-demand verification without
    modifying the core TLS protocol.
- **Protocol Evolution:** Sardar noted that the IETF TLS working group is
  focusing on TLS 1.3, moving away from TLS 1.2 except for critical security
  fixes, and emphasizing post-quantum security.

## **Discussion Highlights**

The audience participation touched on several critical technical and architectural points:

- **Mutual Attestation:** A participant asked about **mutual TLS (mTLS)** where
  both the client and server are attested. Sardar confirmed that the model
  supports both server and client attestation, which is crucial for certain CC
  use cases.
- **Anchoring Trust:** There was a discussion on whether trust should be
  anchored in **hardware-managed evidence** (like direct platform reports) versus
  virtualized sources like **vTPM**. Sardar emphasized that while his model is
  generic, hardware-backed evidence provides stronger guarantees.
- **Post-Handshake vs. Intra-Handshake:** A debate centered on the advantages
  of **post-handshake attestation**. It was noted that this method is less
  intrusive as it doesn't require changes to the base TLS protocol and can work
  across different TLS versions.
- **Channel Binding:** Participants discussed the use of existing **channel
  binding mechanisms** (like RFC 9261) to link attestation evidence to the secure
  channel. Sardar argued for utilizing these established standards rather than
  "reinventing the wheel."

The session concluded with an invitation to further discuss these complex
integration challenges in upcoming IETF meetings and mailing lists,
highlighting that while the "why" of attested TLS is well-understood, the "how"
remains a subject of active debate and standardization effort.
