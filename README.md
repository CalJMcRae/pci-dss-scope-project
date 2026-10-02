# PCI DSS Network Segmentation and Scope Determination

Independent review of whether a company's claimed network segmentation actually holds up
under PCI DSS v4.0, or just looks segmented on a diagram.

**Full write-up:** [`docs/scope-determination.md`](docs/scope-determination.md)

---

## Scenario

**Bluebridge Solutions** is a fictional e-commerce platform provider processing around 2.4
million card transactions a year on behalf of the merchants who run their storefronts on its
platform, which makes it a PCI DSS service provider rather than a merchant. Bluebridge's own
documentation claimed:

1. The cardholder data environment (CDE) is fully segmented from the corporate network
2. Only authorized systems can communicate with the CDE
3. Administrative access requires going through a jump host
4. All CDE traffic is logged and monitored

The task was to test those four claims against the actual network design, not take them at
face value, and produce a scope determination that would survive a QSA's questions.

## What I found

The claims did not hold up. The CDE sits on its own subnet, but it still authenticates
against the corporate Active Directory domain and shares DNS, NTP, patching, backup, and
logging infrastructure with the rest of the business. Each of those shared services is a path
back into the CDE:

- **Active Directory.** A domain compromise on the corporate side hands an attacker
  credentials that also work inside the CDE.
- **Logging.** The logging system ingests masked PAN from CDE systems and forwards it to a
  SIEM that sits outside the CDE. If card data reaches those logs, the SIEM is in scope too.
- **Backups.** The same backup infrastructure serves both the CDE and corporate systems,
  which pulls the entire backup chain into scope.
- **The jump host.** It is the one controlled administrative path into the CDE, which also
  makes it a single point of failure: compromise it, and you have administrative access to
  every CDE system behind it.
- **Corporate workstations.** They can reach the CDE indirectly via VPN and the jump host, so
  the whole workstation subnet carries indirect risk, not just the specific laptops used for
  admin work.

## Conclusion

Segmentation cannot currently be relied upon to reduce PCI DSS scope. Overall risk is rated
**High**. The full write-up documents eight specific gaps, each tied to a PCI DSS v4.0
requirement, a trust-relationship matrix, the questions a QSA would be likely to ask, phased
recommendations (0-30 / 30-90 / 90+ days), and a residual risk statement covering what remains
even after the recommended fixes.

The biggest takeaway: segmentation is an argument you defend with evidence, not a label you
put on a network diagram. Bluebridge's documentation said "segmented" and nobody had actually
tested whether it was.

## What's in this repo

- [`docs/scope-determination.md`](docs/scope-determination.md): the full scope determination
  document, structured the way a real engagement would be: CDE definition, connected and
  security-impacting systems, trust relationship analysis, segmentation gap analysis, QSA
  challenge preparation, phased recommendations, and residual risk.

## Skills demonstrated

PCI DSS v4.0 scope determination · network segmentation analysis · trust relationship mapping
· risk-based gap analysis · QSA-ready documentation · PCI entity classification (merchant vs.
service provider) and its effect on control requirements
