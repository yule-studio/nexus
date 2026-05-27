---
title: "Security / Cryptography — Papers"
kind: knowledge
status: current
created_at: 2026-05-27T12:00:00+09:00
tags: [resources, papers, security, cryptography]
home_hub: resources
---

# Security / Cryptography — Papers

**[[papers|↑ Papers hub]]**

## 1. Cryptography 古典

- ⭐ **New Directions in Cryptography** (Diffie, Hellman, IEEE TIT, 1976) — 공개키 시발
  - 링크: [공식 PDF](https://ee.stanford.edu/~hellman/publications/24.pdf) 🆓

- ⭐ **A Method for Obtaining Digital Signatures and Public-Key Cryptosystems (RSA)** (Rivest, Shamir, Adleman, CACM, 1978)
  - 링크: [MIT](https://people.csail.mit.edu/rivest/Rsapaper.pdf) 🆓

- **Communication Theory of Secrecy Systems** (Claude Shannon, Bell System Tech Journal, 1949) — 정보 이론적 안전성
  - 링크: [PDF](https://www.iacr.org/museum/shannon/shannon45.pdf) 🆓

- **The Discrete Logarithm Problem on Elliptic Curves (ECC)** (Koblitz, Math. of Computation, 1987) — [공식](https://www.ams.org/journals/mcom/1987-48-177/S0025-5718-1987-0866109-5/) 🆓

## 2. Hash / MAC

- **Keying Hash Functions for Message Authentication (HMAC)** (Bellare, Canetti, Krawczyk, CRYPTO, 1996)
  - 링크: [공식 PDF](https://cseweb.ucsd.edu/~mihir/papers/kmd5.pdf) 🆓

- **bcrypt — A Future-Adaptable Password Scheme** (Provos, Mazières, USENIX, 1999)
  - 링크: [USENIX](https://www.usenix.org/legacy/events/usenix99/provos/provos.pdf) 🆓

- **Argon2: the memory-hard function for password hashing and other applications** (Biryukov 외, IEEE Eu S&P, 2016)
  - 링크: [공식](https://www.password-hashing.net/argon2-specs.pdf) 🆓

## 3. TLS / 키 교환

- **The TLS 1.3 Protocol (RFC 8446)** → [[../official-docs/standards-rfc]] §4

- **The Signal Protocol — Double Ratchet Algorithm** (Marlinspike, Perrin, 2016)
  - 링크: [공식 docs](https://signal.org/docs/specifications/doubleratchet/) 🆓

- **WireGuard: Next Generation Kernel Network Tunnel** (Donenfeld, NDSS, 2017)
  - 링크: [공식 PDF](https://www.wireguard.com/papers/wireguard.pdf) 🆓

## 4. Side-channel / 공격

- ⭐ **Spectre — Spectre Attacks: Exploiting Speculative Execution** (Kocher 외, IEEE S&P, 2019)
  - 링크: [공식](https://spectreattack.com/) / [arXiv:1801.01203](https://arxiv.org/abs/1801.01203) 🆓

- ⭐ **Meltdown — Reading Kernel Memory from User Space** (Lipp 외, USENIX Security, 2018)
  - 링크: [공식](https://meltdownattack.com/meltdown.pdf) 🆓

- **Rowhammer — Flipping Bits in Memory Without Accessing Them** (Kim 외, ISCA, 2014)
  - 링크: [공식 PDF](https://users.ece.cmu.edu/~yoonguk/papers/kim-isca14.pdf) 🆓

- **Heartbleed — Heartbleed (CVE-2014-0160 분석)** — [heartbleed.com](https://heartbleed.com/) 🆓

## 5. 인증 / Auth

- ⭐ **OAuth 2.0 RFC 6749** → [[../official-docs/standards-rfc]] §2
- ⭐ **OpenID Connect Core 1.0** → [[../official-docs/standards-rfc]] §2
- **WebAuthn / FIDO2** → [[../official-docs/standards-rfc]] §2

- **Why Johnny Can't Encrypt: A Usability Evaluation of PGP 5.0** (Whitten, Tygar, USENIX Security, 1999)
  - 주제: usability + security 古典
  - 링크: [USENIX](https://www.usenix.org/legacy/events/sec99/full_papers/whitten/whitten.pdf) 🆓

## 6. Web Security

- ⭐ **OWASP Top 10** → [[../official-docs/standards-rfc]] §11

- **XSS / SQL Injection 학술 — Cross-Site Scripting (XSS) Attacks and Defense Mechanisms** (Wassermann, Su, ACM TISSEC, 2008)
  - 링크: [Google Scholar 검색](https://scholar.google.com/scholar?q=XSS+Attacks+and+Defense+Mechanisms+Wassermann)

- **The Tangled Web of Same-Origin Policy** (Adam Barth 외, 2009) — Same-Origin Policy 분석
  - 링크: [Stanford](https://crypto.stanford.edu/websec/cookies/) 🆓

## 7. SE / Software 측 보안

- **Reflections on Trusting Trust** (Ken Thompson, Turing Award Lecture, CACM, 1984)
  - 주제: compiler trojan 古典
  - 링크: [Bell Labs](https://www.cs.cmu.edu/~rdriley/487/papers/Thompson_1984_ReflectionsonTrustingTrust.pdf) 🆓

- **SolarWinds Sunburst — A Comprehensive Look** (다양 industry analysis, 2020+)
  - 링크: [CISA 공식](https://www.cisa.gov/) (검색)

## 8. Supply Chain

- ⭐ **SLSA (Supply-chain Levels for Software Artifacts)** — [slsa.dev](https://slsa.dev/) (spec)
- **in-toto** — [in-toto.io](https://in-toto.io/)
- **Sigstore** — [sigstore.dev](https://www.sigstore.dev/)

## 9. Post-Quantum

- **CRYSTALS-Kyber / Dilithium** — NIST PQC 표준 — [NIST](https://csrc.nist.gov/Projects/post-quantum-cryptography)
- **NIST Post-Quantum Cryptography Standardization** — [공식](https://csrc.nist.gov/projects/post-quantum-cryptography)

## 10. ZKP / 고급 cryptography

- **Zk-SNARKs / Bulletproofs** — 다양 paper
- **Tornado Cash / zk-Rollup 학술** — Ethereum 측 paper 검색

## 11. 학회 / Archive

- ⭐ **IACR ePrint Archive** — Cryptology 무료 paper — [eprint.iacr.org](https://eprint.iacr.org/) 🆓
- **USENIX Security** — [공식](https://www.usenix.org/conferences/byname/108)
- **IEEE S&P (Oakland)** — [공식](https://www.ieee-security.org/TC/SP-Index.html)
- **CCS (ACM)** — [공식](https://www.sigsac.org/ccs.html)
- **NDSS** — [공식](https://www.ndss-symposium.org/)
- **Black Hat / DEF CON** archive → [[../lectures/conferences]] §8

## 12. 관련

- [[papers]] — Papers hub
- [[../books/security/security]] — Security 책
- [[../official-docs/standards-rfc]] §11 — OWASP / NIST / CVE
- [[../articles/topics]] §5 — Security blogs
