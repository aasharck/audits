_this template is copied and modified from https://github.com/pashov/audits/blob/master/Report-template.md. All thanks to pashov!_

# Introduction

A time-boxed security review of the **protocol name** protocol was done by **aashar**, with a focus on the security aspects of the application's implementation.

# Disclaimer

As a solo smart contract auditor, I take smart contract security very seriously. However, it is important to understand that the process of reviewing smart contracts for security vulnerabilities is a time, resource, and expertise bound effort.

While my review is a comprehensive and expert effort to identify as many vulnerabilities as possible, I cannot guarantee the complete absence of vulnerabilities. My review aims to provide the most thorough analysis possible, but it is possible that issues may remain undetected.

It is strongly recommended that subsequent security reviews, bug bounty programs, and on-chain monitoring be conducted to ensure the security of your smart contracts. These measures are essential in maintaining the integrity of your smart contracts and provide an additional layer of security beyond my review.
Thank you for entrusting me with your smart contract security needs.

# About **Aashar**

Aashar ck, is an independent smart contract security researcher. _Add more info_. Reach out on Twitter [@0xaash](https://twitter.com/0xaash)

# About **ProtocolName**

_explanation what the protocol does, some architectural comments, technical documentation_

## Observations

# Threat Model

## Privileged Roles & Actors

## Security Interview

**Q:** What in the protocol has value in the market?

**A:**

**Q:** In what case can the protocol/users lose money?

**A:**

**Q:** What are some ways that an attacker achieves his goals?

**A:**

# Severity classification

| Severity               | Impact: High | Impact: Medium | Impact: Low |
| ---------------------- | ------------ | -------------- | ----------- |
| **Likelihood: High**   | Critical     | High           | Medium      |
| **Likelihood: Medium** | High         | Medium         | Low         |
| **Likelihood: Low**    | Medium       | Low            | Low         |

**Impact** - the technical, economic and reputation damage of a successful attack

**Likelihood** - the chance that a particular vulnerability gets discovered and exploited

**Severity** - the overall criticality of the risk

# Security Assessment Summary

**_review commit hash_ - [fffffffff](url)**

### Scope

The following smart contracts were in scope of the audit:

- `SmartContractName`
- `SmartContractName`

The following number of issues were found, categorized by their severity:

- Critical & High: x issues
- Medium: x issues
- Low: x issues

---

# Findings Summary

| ID     | Title                   | Severity |
| ------ | ----------------------- | -------- |
| [C-01] | Any Critical Title Here | Critical |
| [H-01] | Any High Title Here     | High     |
| [M-01] | Any Medium Title Here   | Medium   |
| [L-01] | Any Low Title Here      | Low      |

# Detailed Findings

# [S-01] {name}

## Severity

**Impact:**

**Likelihood:**

## Description

## Recommendations